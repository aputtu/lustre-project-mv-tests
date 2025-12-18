# Building Robust Web UIs for Lustre Project Quotas

**Technical Synthesis & Implementation Guide**  
*Final Version — December 2025*
Produced by prompting LLMs Gemini and Claude.

**Rationale**: Couple concerns related to Lustre issuing EXDEV, and implementing transaction processing properties (ACID) in a web ui for end users.

---

## 1. The Core Problem: Lustre, Quotas, and Atomicity

When managing files on Lustre with **Project Quotas** enabled, the standard behavior of filesystem operations changes significantly. The central challenge is the tension between **Performance**, **Accounting Accuracy**, and **Operational Safety**.

### The Mechanism: EXDEV (Cross-Device Link)

Lustre uses the POSIX `EXDEV` error code to signal when an atomic rename cannot be performed [1][2]. When project quotas are enabled, Lustre treats directories with different project IDs as separate namespaces for metadata atomicity [3].

| Operation | Scenario | Behavior | Performance |
|-----------|----------|----------|-------------|
| **Rename** | Same Project, Same MDT | Atomic metadata-only rename | ⚡ **Fast** |
| **Move File** | Project A → Project B | **EXDEV Error** — Forces copy + delete | 🐢 **Slow** |
| **Move Directory** | Project A → Project B | **EXDEV Error** — Forces recursive copy + delete | 🐢 **Very Slow** |

> **Important:** The EXDEV behavior occurs because Lustre cannot atomically update project quota accounting across project boundaries — the source project must be debited and destination credited as a transactional unit, which requires data movement.

> **DNE Consideration:** If your Lustre filesystem uses **Distributed Namespace (DNE)** with directories striped across multiple MDTs [4], `os.rename()` can return EXDEV even within the *same* project if source and destination reside on different MDTs.

### How Standard Tools Handle EXDEV

When `mv` or Python's `shutil.move()` encounters EXDEV, they fall back to copy-then-delete [5]:

**Python's `shutil.move()` behavior:**
- First attempts `os.rename()` (atomic, fast)
- On EXDEV failure, copies using `shutil.copy2()` (preserves metadata)
- Then deletes the source

This fallback is **transparent but dangerous** for a Web UI because:

1. **Hidden Latency:** A "Move" operation can silently switch from milliseconds to hours
2. **Non-Atomic:** The copy-then-delete sequence is **not transactional**
3. **Partial State Risk:** If interrupted mid-copy:
   - Files may exist at both source and destination (duplication)
   - Destination may contain incomplete data (corruption)
   - Quota accounting may be inconsistent

### Risk Categories for Web UIs

When a synchronous "Move" becomes a long-running "Copy," you expose the system to these failure modes:

| Risk Category | Description | Example |
|---------------|-------------|---------|
| **Source Mutation** | Files modified or deleted during copy | User B deletes a file while copy is at 50% |
| **Infrastructure Failure** | Process killed, network timeout, browser disconnect | 504 Gateway Timeout leaves orphan `.tmp` files |
| **Quota Exhaustion** | Destination quota exceeded mid-transfer | OSTs stop accepting data while MDT has created zero-byte inodes |
| **I/O Blocking** | Slow or unresponsive storage causes hangs | Network interruption to OSS, or HSM tape recall if applicable |

---

## 2. Best Practices for Working with Lustre

To interact reliably with Lustre project quotas, administrative and backend scripts should follow these patterns.

### Pre-Operation Validation

**Always validate before starting any operation:**

```bash
# 1. Check destination quota capacity (blocks and inodes)
lfs quota -hp <DEST_PROJECT_ID> /mnt/lustre

# 2. Check source size
lfs quota -hp <SRC_PROJECT_ID> /mnt/lustre

# 3. Verify project ID inheritance on destination parent
lfs project -d /destination/parent
```

**Key validation checks:**
- **Block capacity:** Destination free space > Source usage
- **Inode capacity:** Destination free inodes > Source file count (often overlooked)
- **Inheritance flag:** Destination parent should have the `P` (inherit) flag set

> **Quota Cache Note:** Lustre clients receive "granted cache" for writes, and quota updates propagate asynchronously across OSTs [6]. Brief inconsistencies are normal; always include a safety margin in capacity checks.

### Directory Inheritance Setup

Ensure destination directories inherit the correct project ID. This guarantees that files copied into them automatically adopt the correct project ID for quota accounting [7].

```bash
# Create destination with project ID and inheritance flag
mkdir -p /destination/target
lfs project -sp <PROJECT_ID> /destination/target
```

The flags:
- `-s` : Set the inheritance flag (critical for new files)
- `-p` : Set the project ID
- `-r` : Apply recursively (use only on existing trees)

**Verify the setup:**
```bash
$ lfs project -d /destination/target
<PROJECT_ID> P /destination/target
#            ^ 'P' indicates inheritance is set
```

Without the `P` (Project Inheritance) flag, files moved into a directory may retain their old project ID or default to the user's ID, defeating quota management entirely.

### Avoiding the "Pre-Change Source" Anti-Pattern

⚠️ **Do NOT modify the source's project ID before moving:**

```bash
# DANGEROUS - Do not do this:
lfs project -rp <DEST_ID> /source/directory  # Changes quota accounting immediately
mv /source/directory /destination/           # If this fails, quota state is corrupted
```

**Why this is dangerous:**

Changing a source directory's Project ID (`lfs project -rp`) before moving it is equivalent to **writing to a production database without a backup**. If the move fails halfway:

1. The source data is now misattributed to the wrong project
2. Restoring the original quota state requires a recursive walk
3. That recursive restoration may itself fail due to quota issues on the original project
4. Race conditions: new files created during `lfs project -r` may have inconsistent IDs

**Safe alternative:** Always copy to a properly configured destination, then delete source after verification.

### Quota Enforcement Caveats

Be aware of these Lustre quota behaviors [6]:

1. **Root bypass:** Quota limits are not enforced for the root user or processes using `sudo`
2. **Grace periods:** Soft limits allow temporary overages; check both soft and hard limits
3. **Granted cache:** Clients receive write grants that may allow brief overages of hard limits
4. **Distributed lag:** Quota updates propagate asynchronously across OSTs

---

## 3. Implementation Guide: Staged Commit with Verification

A Web UI cannot rely on simple POSIX calls for cross-project operations. You must implement an **Asynchronous Task Engine** that provides crash recovery and verification.

> **Note on Terminology:** This pattern provides **eventual consistency** and **crash recovery**, not true ACID semantics [8]. Filesystem operations cannot provide the same isolation guarantees as database transactions because the filesystem is a shared namespace subject to external modification.

### Architecture Overview

Instead of moving files directly to the final location, use a **staged commit** approach with a shadow copy:

```
┌─────────────────────────────────────┐
│  STAGED COMMIT WORKFLOW             │
├─────────────────────────────────────┤
│                                     │
│  ┌───────────────────────────────┐  │
│  │ 1. VALIDATE                   │  │
│  │    • Check quota capacity     │  │
│  │    • Verify destination setup │  │
│  │    • Detect collisions        │  │
│  └───────────────┬───────────────┘  │
│                  ▼                  │
│  ┌───────────────────────────────┐  │
│  │ 2. LOCK                       │  │
│  │    • Mark task "running"      │  │
│  │    • Prevent user interference│  │
│  └───────────────┬───────────────┘  │
│                  ▼                  │
│  ┌───────────────────────────────┐  │
│  │ 3. STAGE                      │  │
│  │    • Create .staging_<id> dir │  │
│  │    • Set project inheritance  │  │
│  │    • Preserve striping config │  │
│  │    • Copy data to staging     │  │
│  └───────────────┬───────────────┘  │
│                  ▼                  │
│  ┌───────────────────────────────┐  │
│  │ 4. VERIFY                     │  │
│  │    • Compare file counts      │  │
│  │    • Check size AND blocks    │  │
│  │    • Detect source changes    │  │
│  └───────────────┬───────────────┘  │
│                  ▼                  │
│  ┌───────────────────────────────┐  │
│  │ 5. COMMIT                     │  │
│  │    • Atomic rename to final   │  │
│  │    • fsync parent directory   │  │
│  │    • Delete source            │  │
│  │    • Release lock             │  │
│  └───────────────────────────────┘  │
│                                     │
│  ON FAILURE AT ANY STAGE:           │
│  → Cleanup .staging_* directory     │
│  → Release lock                     │
│  → Report error to user             │
│                                     │
└─────────────────────────────────────┘
```

### The 5-Phase Workflow

#### Phase 1: Validation Gate

Perform all pre-flight checks before any data movement:

```python
def validate_move(source_path, dest_parent, dest_project_id):
    """
    Returns (success: bool, error_message: str | None)
    """
    # Check source exists
    if not os.path.exists(source_path):
        return False, f"Source does not exist: {source_path}"
    
    # Get source statistics
    source_stats = get_directory_stats(source_path)  # bytes, file_count
    
    # Check destination quota (with safety margin)
    dest_quota = get_project_quota(dest_project_id)  # lfs quota -hp ...
    safety_margin = 1.05  # 5% buffer for quota cache lag
    
    if source_stats.bytes * safety_margin > dest_quota.free_bytes:
        return False, f"Insufficient space: need {source_stats.bytes}, have {dest_quota.free_bytes}"
    
    if source_stats.file_count * safety_margin > dest_quota.free_inodes:
        return False, f"Insufficient inodes: need {source_stats.file_count}, have {dest_quota.free_inodes}"
    
    # Check destination parent has correct project inheritance
    parent_info = get_project_info(dest_parent)  # lfs project -d
    if parent_info.project_id != dest_project_id:
        return False, f"Destination parent has wrong project ID: {parent_info.project_id}"
    if not parent_info.has_inheritance:
        return False, f"Destination parent missing inheritance flag"
    
    # Check for collision at destination
    final_dest = os.path.join(dest_parent, os.path.basename(source_path))
    if os.path.exists(final_dest):
        return False, f"Destination already exists: {final_dest}"
    
    return True, None
```

#### Phase 2: Isolation (Task Locking)

Prevent concurrent operations and user interference:

```python
def acquire_task_lock(source_path, task_id, db_connection):
    """
    Mark operation as in-progress in your task database.
    This prevents the UI from allowing conflicting operations.
    """
    with db_connection.transaction():
        existing = db_connection.query(
            "SELECT task_id FROM active_tasks WHERE path = %s FOR UPDATE",
            [source_path]
        )
        if existing:
            raise ConflictError(f"Path locked by task {existing.task_id}")
        
        db_connection.execute(
            "INSERT INTO active_tasks (path, task_id, started_at, status) "
            "VALUES (%s, %s, NOW(), 'running')",
            [source_path, task_id]
        )
```

#### Phase 3: Durability (Shadow Copy with Stripe Preservation)

Copy to a staging location within the destination project, preserving Lustre-specific attributes [9]:

```python
import uuid
import shutil
import subprocess
import os

def stage_copy(source_path, dest_parent, dest_project_id):
    """
    Copy source to a hidden staging directory, preserving Lustre striping.
    
    IMPORTANT: The staging directory MUST be in the same parent as the
    final destination to ensure atomic rename works (same MDT in DNE setups).
    
    Returns (staging_path, staged_content_path)
    """
    # Create unique staging directory in the SAME parent as destination
    staging_name = f".staging_{uuid.uuid4().hex}"
    staging_path = os.path.join(dest_parent, staging_name)
    
    # Create staging dir with correct project ID and inheritance
    os.makedirs(staging_path)
    subprocess.run(
        ['lfs', 'project', '-sp', str(dest_project_id), staging_path],
        check=True
    )
    
    source_name = os.path.basename(source_path)
    staged_dest = os.path.join(staging_path, source_name)
    
    if os.path.isdir(source_path):
        # For directories, copy tree with stripe preservation
        copy_tree_with_striping(source_path, staged_dest)
    else:
        # For single files, preserve striping then copy
        copy_file_with_striping(source_path, staged_dest)
    
    return staging_path, staged_dest


def get_stripe_info(path):
    """Get Lustre striping configuration for a file."""
    result = subprocess.run(
        ['lfs', 'getstripe', '-c', '-S', path],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        return None
    
    lines = result.stdout.strip().split('\n')
    # Parse stripe count and size from output
    stripe_count = None
    stripe_size = None
    for line in lines:
        if 'lmm_stripe_count' in line:
            stripe_count = int(line.split(':')[1].strip())
        elif 'lmm_stripe_size' in line:
            stripe_size = int(line.split(':')[1].strip())
    
    return {'count': stripe_count, 'size': stripe_size}


def copy_file_with_striping(source, dest):
    """
    Copy a single file, preserving Lustre striping configuration.
    """
    stripe_info = get_stripe_info(source)
    
    # Pre-create destination with matching stripe configuration
    if stripe_info and stripe_info['count'] and stripe_info['count'] > 1:
        cmd = ['lfs', 'setstripe', '-c', str(stripe_info['count'])]
        if stripe_info['size']:
            cmd.extend(['-S', str(stripe_info['size'])])
        cmd.append(dest)
        subprocess.run(cmd, check=True)
        
        # Now copy data into the pre-striped file
        shutil.copyfile(source, dest)
        shutil.copystat(source, dest)
    else:
        # Default striping is fine, use standard copy
        shutil.copy2(source, dest)


def copy_tree_with_striping(source, dest):
    """
    Recursively copy a directory tree, preserving Lustre striping.
    """
    os.makedirs(dest, exist_ok=True)
    shutil.copystat(source, dest)
    
    with os.scandir(source) as entries:
        for entry in entries:
            src_path = entry.path
            dst_path = os.path.join(dest, entry.name)
            
            if entry.is_dir(follow_symlinks=False):
                copy_tree_with_striping(src_path, dst_path)
            elif entry.is_symlink():
                link_target = os.readlink(src_path)
                os.symlink(link_target, dst_path)
            else:
                copy_file_with_striping(src_path, dst_path)
```

**Why staging works:**
- If the worker crashes, the real destination is untouched
- Staging directory can be identified and cleaned by a janitor process
- The final "swap" into place is a fast atomic rename
- Creating staging in the same parent ensures same-MDT placement for DNE

#### Phase 4: Verification (Integrity Check with Sparse File Support)

Verify the copy completed correctly and source hasn't changed. Compare both `st_size` and `st_blocks` to handle sparse files correctly [10]:

```python
from dataclasses import dataclass
from typing import Dict, Optional, Tuple
import os

@dataclass
class FileInfo:
    size: int       # Logical size (st_size)
    blocks: int     # Actual blocks allocated (st_blocks)
    mtime: float    # Modification time


def verify_copy(source_path: str, staged_path: str) -> Tuple[bool, Optional[str]]:
    """
    Compare source and staged copy, including sparse file validation.
    Returns (success, error_message)
    """
    source_files = collect_file_info(source_path)
    staged_files = collect_file_info(staged_path)
    
    # Check file count matches
    if len(source_files) != len(staged_files):
        return False, (
            f"File count mismatch: source={len(source_files)}, "
            f"staged={len(staged_files)}"
        )
    
    # Check each file
    for rel_path, source_info in source_files.items():
        staged_info = staged_files.get(rel_path)
        
        if staged_info is None:
            return False, f"Missing file in staged copy: {rel_path}"
        
        # Compare logical size
        if source_info.size != staged_info.size:
            return False, f"Size mismatch for {rel_path}"
        
        # Compare actual blocks (catches sparse file issues)
        # Allow small variance due to filesystem block size differences
        block_tolerance = 8  # 4KB tolerance
        if abs(source_info.blocks - staged_info.blocks) > block_tolerance:
            return False, (
                f"Block count mismatch for {rel_path}: "
                f"source={source_info.blocks}, staged={staged_info.blocks} "
                f"(possible sparse file corruption)"
            )
        
        # Compare mtime (detect source modification during copy)
        if source_info.mtime != staged_info.mtime:
            return False, f"Source was modified during copy: {rel_path}"
    
    return True, None


def collect_file_info(root_path: str) -> Dict[str, FileInfo]:
    """
    Collect file metadata for comparison.
    Uses os.scandir() for efficient metadata collection on large directories.
    """
    files = {}
    
    def scan_recursive(path: str, prefix: str = ""):
        try:
            with os.scandir(path) as it:
                for entry in it:
                    rel_path = os.path.join(prefix, entry.name) if prefix else entry.name
                    
                    if entry.is_file(follow_symlinks=False):
                        stat = entry.stat(follow_symlinks=False)
                        files[rel_path] = FileInfo(
                            size=stat.st_size,
                            blocks=stat.st_blocks,
                            mtime=stat.st_mtime,
                        )
                    elif entry.is_dir(follow_symlinks=False):
                        scan_recursive(entry.path, rel_path)
                    # Symlinks: we could track them separately if needed
        except PermissionError as e:
            raise VerificationError(f"Permission denied scanning {path}: {e}")
    
    scan_recursive(root_path)
    return files
```

#### Phase 5: Commit (Atomic Swap with Durable Rename)

Atomically move staged data to final location and clean up. The rename is atomic per POSIX [1], but durability requires fsync on the parent directory [11]:

```python
def durable_rename(src: str, dst: str):
    """
    Perform a rename that is guaranteed durable after return.
    The rename is only safe after the parent directory metadata is flushed.
    """
    os.rename(src, dst)
    
    # Sync the directory containing the new link
    parent_dir = os.path.dirname(dst)
    dfd = os.open(parent_dir, os.O_RDONLY | os.O_DIRECTORY)
    try:
        os.fsync(dfd)
    finally:
        os.close(dfd)


def commit_move(staging_path: str, staged_content: str, 
                final_dest: str, source_path: str):
    """
    Atomically complete the move operation.
    """
    # Atomic rename from staging to final destination
    # This works because staging_path and final_dest are in the same parent
    durable_rename(staged_content, final_dest)
    
    # Clean up empty staging directory
    os.rmdir(staging_path)
    
    # Only now delete the source
    # If we crash here, we have duplication (safe) not data loss
    if os.path.isdir(source_path):
        shutil.rmtree(source_path)
    else:
        os.unlink(source_path)
    
    # Sync source parent directory to ensure delete is durable
    source_parent = os.path.dirname(source_path)
    dfd = os.open(source_parent, os.O_RDONLY | os.O_DIRECTORY)
    try:
        os.fsync(dfd)
    finally:
        os.close(dfd)
```

> **Critical:** The `os.fsync()` calls on parent directories ensure the rename and delete operations are durable [11]. Without these, a system crash could result in the operations being lost even though they appeared to complete. In Linux, a `rename()` is only guaranteed to be durable after the parent directory's metadata is flushed.

### Technology Stack Recommendations

#### Task Queue

Use a distributed task queue (such as **Celery** for Python or **BullMQ** for Node.js) with appropriate settings for long-running filesystem tasks [12]:

**Key configuration requirements:**
- **Late acknowledgment:** Don't acknowledge task completion until it actually completes
- **Requeue on worker loss:** If a worker dies, the task should be retried
- **Reasonable timeouts:** Set both soft and hard time limits appropriate for your data sizes
- **Idempotency awareness:** Design tasks to handle being run multiple times safely

```python
# Example Celery task configuration (adapt for your task queue)
@task_queue.task(
    acks_late=True,              # Don't ack until task completes
    reject_on_worker_lost=True,  # Requeue if worker dies
    max_retries=3,
    default_retry_delay=60,
    time_limit=7200,             # Hard limit: 2 hours
    soft_time_limit=6900,        # Soft limit: warn at 1:55
)
def move_directory_task(source_path, dest_parent, dest_project_id, task_id):
    """
    Task for moving directories across project boundaries.
    """
    try:
        # Phase 1: Validate
        success, error = validate_move(source_path, dest_parent, dest_project_id)
        if not success:
            return {'status': 'failed', 'error': error}
        
        # Phase 2: Lock
        acquire_task_lock(source_path, task_id)
        
        try:
            # Phase 3: Stage (with stripe preservation)
            update_task_status(task_id, 'copying')
            staging_path, staged_content = stage_copy(
                source_path, dest_parent, dest_project_id
            )
            
            # Phase 4: Verify
            update_task_status(task_id, 'verifying')
            success, error = verify_copy(source_path, staged_content)
            if not success:
                shutil.rmtree(staging_path)  # Cleanup
                return {'status': 'failed', 'error': error}
            
            # Phase 5: Commit
            update_task_status(task_id, 'committing')
            final_dest = os.path.join(dest_parent, os.path.basename(source_path))
            commit_move(staging_path, staged_content, final_dest, source_path)
            
            return {'status': 'success', 'destination': final_dest}
            
        finally:
            release_task_lock(task_id)
            
    except SoftTimeLimitExceeded:
        cleanup_staging(task_id)
        return {'status': 'failed', 'error': 'Operation timed out'}
```

#### State Management

Track operation progress for UI feedback. Your database schema should include:

```sql
CREATE TABLE move_operations (
    task_id         UUID PRIMARY KEY,
    source_path     VARCHAR(4096) NOT NULL,
    dest_path       VARCHAR(4096) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    progress_bytes  BIGINT DEFAULT 0,
    total_bytes     BIGINT DEFAULT 0,
    error_message   TEXT,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

-- Status values: pending, validating, copying, verifying, committing, completed, failed

CREATE INDEX idx_move_operations_status ON move_operations(status);
CREATE INDEX idx_move_operations_source ON move_operations(source_path);
```

#### The Janitor Process

A scheduled job to clean up orphaned staging directories:

```python
from datetime import datetime, timedelta
import logging

log = logging.getLogger(__name__)

def cleanup_orphaned_staging(project_directories, db_connection, max_age_hours=24):
    """
    Run periodically (e.g., hourly) to clean up failed operations.
    """
    threshold = datetime.now() - timedelta(hours=max_age_hours)
    
    for project_dir in project_directories:
        try:
            with os.scandir(project_dir) as entries:
                for entry in entries:
                    if entry.name.startswith('.staging_') and entry.is_dir():
                        mtime = datetime.fromtimestamp(entry.stat().st_mtime)
                        
                        if mtime < threshold:
                            # Extract task_id from directory name
                            task_id = entry.name.replace('.staging_', '')
                            
                            # Check if there's an active task
                            active = db_connection.query(
                                "SELECT 1 FROM active_tasks "
                                "WHERE task_id = %s AND status = 'running'",
                                [task_id]
                            )
                            
                            if not active:
                                log.warning(f"Cleaning orphaned staging: {entry.path}")
                                shutil.rmtree(entry.path)
        except OSError as e:
            log.error(f"Error scanning {project_dir}: {e}")
```

### Implementation Risks Summary

| Component | Risk | Mitigation |
|-----------|------|------------|
| **Quota** | Client-side granted cache causes brief overages | Include 5% safety margin in capacity checks |
| **Permissions** | `shutil.copy2` fails to preserve ownership | Ensure worker has `CAP_CHOWN` or runs privileged |
| **Large Trees** | Slow metadata collection | Use `os.scandir()` instead of `pathlib.rglob` [10] |
| **Sparse Files** | Size matches but data missing | Compare both `st_size` and `st_blocks` [10] |
| **Striping** | Default striping applied to copies | Pre-create files with `lfs setstripe` before copy [9] |
| **DNE** | Cross-MDT rename fails with EXDEV | Create staging in same parent as destination [4] |

### Existing Solutions

If building a custom solution is too complex, consider these alternatives:

| Solution | Use Case | Key Features |
|----------|----------|--------------|
| **Globus** | Large-scale data transfer | Async transfer, verification, retries, web UI |
| **Starfish** | Storage management | Policy-aware moves, metadata tracking |
| **Robinhood** | Lustre policy engine | Policy-based data movement, reporting |

---

## 4. Optional: HSM Integration

If your Lustre installation uses **Hierarchical Storage Management (HSM)** to archive files to tape or other backends, additional considerations apply [13].

### HSM-Specific Risks

Files in the `released` state (archived to tape, no local copy) will cause blocking behavior:
- `open()` calls block indefinitely waiting for tape recall
- Copy operations can hang for minutes to hours
- No timeout mechanism exists at the filesystem level

### Pre-Operation HSM Check

Add HSM state validation to your Phase 1 checks:

```bash
# Check HSM state for a single file
lfs hsm_state /path/to/file

# Find files that would require tape recall
lfs find /path -hsm released
```

```python
def check_hsm_released(source_path: str) -> list:
    """
    Check if any files are in 'released' state (on tape only).
    Returns list of released file paths.
    """
    result = subprocess.run(
        ['lfs', 'find', source_path, '-hsm', 'released'],
        capture_output=True, text=True
    )
    released_files = [f for f in result.stdout.strip().split('\n') if f]
    return released_files

# In validation phase:
released_files = check_hsm_released(source_path)
if released_files:
    return False, (
        f"Files must be recalled from tape first: {len(released_files)} files. "
        f"Use 'lfs hsm_restore' to recall them."
    )
```

### HSM Recall Before Move

If you need to support moving HSM-archived data, trigger recall first and wait for completion:

```bash
# Recall a file from tape
lfs hsm_restore /path/to/file

# Check recall progress (returns empty when complete)
lfs hsm_action /path/to/file
```

---

## 5. Summary of Key Points

1. **EXDEV is the signal:** When Lustre returns EXDEV, you must copy data — there's no workaround for cross-project atomic renames.

2. **Never modify source project IDs:** Always copy to a correctly-configured destination; modifying the source corrupts quota state on failure.

3. **Use staged commits:** Copy to a hidden staging directory in the same parent as the destination, verify, then atomically rename to the final location.

4. **Preserve Lustre striping:** Standard copy tools don't preserve striping; use `lfs setstripe` to pre-create files with matching configuration.

5. **Verify thoroughly:** Compare both `st_size` and `st_blocks` to catch sparse file issues.

6. **fsync parent directories:** Atomic rename is only durable after syncing the parent directory metadata.

7. **Design for failure:** Workers can die at any point. Ensure every state is either recoverable or cleanly aborted.

8. **Account for DNE:** In multi-MDT environments, ensure staging and destination are in the same parent directory.

---

## Appendix A: Quick Reference Commands

```bash
# Check project ID and inheritance flag
lfs project -d /path/to/directory

# Set project ID with inheritance (for new directories)
lfs project -sp <PROJECT_ID> /path/to/directory

# Apply project ID recursively (existing trees only)
lfs project -rp <PROJECT_ID> /path/to/directory

# Check quota usage
lfs quota -hp <PROJECT_ID> /mnt/lustre

# Check file striping configuration
lfs getstripe /path/to/file
lfs getstripe -c /path/to/file    # Just stripe count

# Set striping on new file
lfs setstripe -c <COUNT> -S <SIZE> /path/to/new_file

# HSM commands (if HSM is enabled)
lfs hsm_state /path/to/file       # Check archive state
lfs find /path -hsm released      # Find files needing recall
lfs hsm_restore /path/to/file     # Recall from tape
lfs hsm_action /path/to/file      # Check recall progress
```

---

## Appendix B: References

### Canonical Sources

[1] **The Open Group Base Specifications Issue 7 (POSIX.1-2017)** — `rename()` function specification.  
https://pubs.opengroup.org/onlinepubs/9699919799/functions/rename.html

[2] **Linux Programmer's Manual** — `rename(2)` man page, including EXDEV behavior.  
https://man7.org/linux/man-pages/man2/rename.2.html

[3] **Lustre Operations Manual** — Comprehensive administration guide covering quotas, striping, and filesystem operations.  
https://doc.lustre.org/lustre_manual.xhtml

[4] **Lustre Wiki: Distributed Namespace (DNE)** — Architecture documentation for multi-MDT directory striping.  
https://wiki.lustre.org/Distributed_Namespace_and_Remote_Directories

[5] **Python Documentation: shutil — High-level file operations** — `shutil.move()` behavior including EXDEV fallback to copy+delete.  
https://docs.python.org/3/library/shutil.html

[6] **Lustre Wiki: Quota Troubleshooting** — Quota enforcement, space accounting, and project quota configuration for Lustre 2.15+.  
https://wiki.lustre.org/Lustre_Quota_Troubleshooting

[7] **Google Cloud Managed Lustre: File System Quotas** — Project quota setup including `-s` (inheritance) and `-p` flags.  
https://docs.cloud.google.com/managed-lustre/docs/file-system-quotas

[8] **Wright et al., "Extending ACID Semantics to the File System"** — Academic paper on filesystem transaction limitations, Stony Brook University FSL.  
https://www.fsl.cs.sunysb.edu/docs/amino-tr/amino.pdf

[9] **Lustre Wiki: Understanding Lustre Internals** — Striping architecture, FIDs, and Layout Extended Attributes.  
https://wiki.lustre.org/Understanding_Lustre_Internals

[10] **Python Documentation: os.scandir()** — Efficient directory iteration with cached stat results.  
https://docs.python.org/3/library/os.html#os.scandir

[11] **LWN.net: "A way to do atomic writes"** — Discussion of fsync requirements for durable rename operations.  
https://lwn.net/Articles/789600/

[12] **Celery Documentation: Tasks** — Task configuration including `acks_late`, idempotency, and retry behavior.  
https://docs.celeryq.dev/en/stable/userguide/tasks.html

[13] **Lustre Operations Manual: Hierarchical Storage Management (HSM)** — HSM states, `lfs hsm_*` commands, and Copytool architecture.  
https://doc.lustre.org/lustre_manual.xhtml#lustrehsm

### Additional Resources

- **AWS FSx for Lustre: Using Storage Quotas** — Project quota examples for managed Lustre.  
  https://docs.aws.amazon.com/fsx/latest/LustreGuide/lustre-quotas.html

- **Azure Managed Lustre: Use Quotas** — Quota configuration and enforcement behavior.  
  https://learn.microsoft.com/en-us/azure/azure-managed-lustre/lustre-quotas

- **Lustre Mailing List Archives** — Community discussions on project quota EXDEV behavior.  
  http://lists.lustre.org/pipermail/lustre-discuss-lustre.org/

- **Wikipedia: Atomicity (database systems)** — ACID properties and filesystem atomicity limitations.  
  https://en.wikipedia.org/wiki/Atomicity_(database_systems)
