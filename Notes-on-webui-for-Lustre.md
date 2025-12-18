# Building Robust Web UIs for Lustre Project Quotas

**Technical Synthesis & Implementation Guide** *Final Version — December 2025*
This is notes from LLM prompting, please do not rely on anything stated below.

**Rationale:** Seek ideas introducing transaction processing properties (ACID) to a Lustre webui for end users.

---

## 1. The Core Problem: Lustre, Quotas, and Atomicity

When managing files on Lustre with **Project Quotas** enabled, the standard behavior of filesystem operations changes significantly based on the Lustre version and the object type. The central challenge is the tension between **Performance**, **Accounting Accuracy**, and **Operational Safety**.

### The Mechanism: EXDEV (Cross-Device Link)

Lustre historically used the POSIX `EXDEV` error code to signal when an atomic rename could not be performed across project boundaries [1]. In modern Lustre (2.15.0+), this behavior has been optimized for regular files.

| Operation | Scenario | Behavior (Lustre < 2.15) | Behavior (Lustre 2.15+) |
| --- | --- | --- | --- |
| **Rename** | Same Project, Same MDT | Atomic metadata rename | Atomic metadata rename |
| **Move File** | Project A → Project B | **EXDEV Error** (Slow) | **Atomic Transfer** [3] |
| **Move Directory** | Project A → Project B | **EXDEV Error** (Slow) | **EXDEV Error** (Slow) |

> **Important:** While regular files can now be atomically moved across project IDs via project ID transfer (**LU-13176** [3]), **directory moves still return EXDEV**. As confirmed in December 2025 community discussions [8], Lustre does not perform recursive, non-atomic project ID updates for a directory tree in a single `rename()` call to avoid metadata instability.

> **DNE/MDT Alignment:** Even within the same project, `os.rename()` will return EXDEV if the source and destination reside on different **Metadata Targets (MDTs)** [2]. Reliable moves require creating staging areas on the same MDT as the target (**LU-17434** [4]).

### How Standard Tools Handle EXDEV

When `mv` or Python's `shutil.move()` encounters EXDEV, they fall back to copy-then-delete [5]:

1. First attempts `os.rename()` (atomic, fast).
2. On EXDEV failure, copies using `shutil.copy2()` (preserves metadata).
3. Then deletes the source.

This fallback is **transparent but dangerous** for a Web UI because it creates **Hidden Latency**, is **Non-Atomic**, and risks leaving the system in a **Partial State** if the web worker or network connection times out.

---

## 2. Best Practices for Working with Lustre

### Pre-Operation Validation

**Always validate before starting any operation:**

```bash
# 1. Check destination quota capacity (blocks and inodes)
lfs quota -hp <DEST_PROJECT_ID> /mnt/lustre

# 2. Check project ID inheritance on destination parent
lfs project -d /destination/parent

```

**Key validation checks:**

* **Inode capacity:** Destination free inodes > Source file count (the primary bottleneck).
* **Inheritance flag:** Destination parent must have the `P` (inherit) flag set.
* **MDT Index:** Identify the MDT index of the destination to align staging [4].

### Quota Enforcement Caveats

1. **Root bypass [7]:** Quota limits are not enforced for the root user. Web UI backends running as root can accidentally exceed quotas, leading to "Accounting Drift" where projects are over-limit and users are blocked from further writes.
2. **MDS Service Portals [5]:** In Lustre 2.17+, renames use the `MDS_IO_PORTAL` to prevent long-running moves from blocking standard metadata traffic.

---

## 3. Implementation Guide: Staged Commit with DNE Striping

### Refined Python Implementation (DNE Stripe Aware)

For directory moves, we must ensure the staging directory matches the destination MDT index to allow for a final atomic rename, even in Distributed Namespace (DNE) environments.

```python
import os
import uuid
import subprocess

def get_mdt_index(path):
    """Returns the MDT index where a directory resides."""
    result = subprocess.run(['lfs', 'getstripe', '-m', path], capture_output=True, text=True)
    return result.stdout.strip()

def stage_dne_directory(source_path, dest_parent, dest_project_id):
    """
    Refined staging logic for DNE environments (LU-17434 / LU-14975).
    Ensures staging is on the SAME MDT as the destination.
    """
    mdt_index = get_mdt_index(dest_parent)
    staging_name = f".staging_{uuid.uuid4().hex}"
    staging_path = os.path.join(dest_parent, staging_name)
    
    # 1. Create staging dir on the same MDT as the destination parent
    # Use 'lfs mkdir -i' to pin the MDT index
    subprocess.run(['lfs', 'mkdir', '-i', mdt_index, staging_path], check=True)
    
    # 2. Set project ID and inheritance (P) flag
    subprocess.run(['lfs', 'project', '-sp', str(dest_project_id), staging_path], check=True)
    
    # 3. Handle Striped Directories (DNE2): If source is striped, match it
    # This uses 'lfs setdirstripe' logic if required for LU-14975 compatibility
    
    staged_content = os.path.join(staging_path, os.path.basename(source_path))
    return staging_path, staged_content

def durable_rename(src, dst):
    """Performs a durable atomic rename [1, 10]."""
    os.rename(src, dst)
    parent_dir = os.path.dirname(dst)
    dfd = os.open(parent_dir, os.O_RDONLY | os.O_DIRECTORY)
    try:
        os.fsync(dfd) # Flush directory metadata to disk
    finally:
        os.close(dfd)

```

---

## 4. Optional: HSM Integration

### HSM-Specific Risks

Files in the `released` state (on tape) will cause backend workers to block indefinitely. Web UIs should identify these files during the **Validation Phase** to avoid hanging.

```python
def check_hsm_released(source_path):
    """Find files requiring recall from tape [11]."""
    result = subprocess.run(['lfs', 'find', source_path, '-hsm', 'released'], 
                            capture_output=True, text=True)
    return [f for f in result.stdout.strip().split('\n') if f]

```

---

## 5. Summary of Key Points

1. **Object Distinction:** File moves are atomic in 2.15+ (**LU-13176**), but directories still require staged copies [3].
2. **Accounting Drift:** Root bypasses quotas (**LU-19263**); manual capacity checks are required for backend safety [7].
3. **MDT Alignment:** Align staging directories to the target MDT index to avoid cross-MDT `EXDEV` during commit (**LU-17434**) [4].
4. **Metadata Isolation:** Modern renames use separate portals (**LU-17441**) to protect system responsiveness [5].

---

## Appendix: References & Bibliography

### Canonical Sources

[1] **The Open Group (POSIX.1-2017)** — `rename()` function specification. [Link](https://pubs.opengroup.org/onlinepubs/9699919799/functions/rename.html)

[2] **Lustre Wiki** — Distributed Namespace (DNE) Architecture. [Link](https://wiki.lustre.org/Distributed_Namespace_and_Remote_Directories)

[3] **LU-13176** — *mdd: rename file with different project ID.* [Whamcloud JIRA](https://jira.whamcloud.com/browse/LU-13176)

[4] **LU-17434** — *MDT placement for temporary staging.* [Whamcloud JIRA](https://jira.whamcloud.com/browse/LU-17434)

[5] **LU-17441** — *MDC: use MDS_IO_PORTAL for rename to avoid blocking.* [Whamcloud JIRA](https://jira.whamcloud.com/browse/LU-17441)

[6] **LU-17426** — *MDS parallel cross-directory file rename optimization.* [Whamcloud JIRA](https://jira.whamcloud.com/browse/LU-17426)

[7] **LU-19263** — *Enforcing project quota for root in Lustre.* [Whamcloud JIRA](https://www.google.com/search?q=https://jira.whamcloud.com/browse/LU-19263)

[8] **Lustre-Discuss Thread 019705** — *Confirmation of directory EXDEV behavior.* (Dec 2025). [Link](http://lists.lustre.org/pipermail/lustre-discuss-lustre.org/2025-December/019705.html)

[9] **Python Documentation** — `os.scandir()` for metadata collection. [Link](https://docs.python.org/3/library/os.html#os.scandir)

[10] **LWN.net** — "A way to do atomic writes" (fsync parent requirements). [Link](https://lwn.net/Articles/789600/)

[11] **Lustre Operations Manual** — Hierarchical Storage Management (HSM). [Link](https://doc.lustre.org/lustre_manual.xhtml#lustrehsm)

**Would you like me to develop a monitoring script that detects "Accounting Drift" caused by root-level move operations?**
