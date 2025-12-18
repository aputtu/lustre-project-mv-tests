
# Building Robust Web UIs for Lustre Project Quotas

**Application-Level Transaction Design for Lustre File Operations** 
**Revised, Evidence-Constrained Edition — December 2025**

**Warning**: This is only ideas bounced against LLMs. Please do **NOT** rely on anything stated below.

---

## Scope and Motivation

The primary objective is to enable **end users to perform filesystem operations safely through a Web UI** on a Lustre-backed system. At scale, these operations diverge into two distinct architectural paths:

1. **The Transactional Path (Small/Medium Data):** Prioritizes ACID-like safety via staging and verification.
2. **The Administrative Path (Large/Massive Data):** Prioritizes performance via metadata re-tagging, requiring strict application-level guardrails.

On Lustre, the gap between user expectations and filesystem guarantees is widened by project quota boundaries, distributed metadata (DNE), and backend services running with elevated privileges. **Consistency must be implemented at the application layer.**

---

## 1. The Core Problem: Lustre, Project Quotas, and Atomicity

Lustre follows the POSIX model strictly, returning `EXDEV` when an atomic rename cannot be guaranteed across accounting or namespace boundaries.

Historically, **project ID boundaries** were absolute barriers for atomic operations. While Lustre 2.15+ introduced file-level optimizations (**LU-13176**), **directory moves remain non-atomic**. A recursive project ID update for a directory tree is a multi-step mutation that can fail mid-process, leaving the filesystem in a partial state.

---

## 2. Choosing the Implementation Path

| Feature | **Path A: Staged Commit (Transactional)** | **Path B: Direct Migration (Administrative)** |
| --- | --- | --- |
| **Primary Tool** | `shutil.copy2` / `os.rename` | `lfs project -rp` |
| **Data Size** | Small to Medium (< 500GB) | Large to Massive (> 500GB / Millions of files) |
| **Safety** | High (Original data is untouched) | Low (In-place mutation) |
| **Atomicity** | Achieved via Staging/Swap | None (Iterative Metadata Walk) |
| **I/O Cost** | High (Full Data Read/Write) | Low (Metadata only) |

---

## 3. Path A: The Transactional Workflow (Small/Medium Data)

For operations where data movement is feasible, use the **5-Phase Workflow**:

1. **Validation Gate:** Verify destination quota and MDT index. Note that `root` backends bypass quotas (**LU-19263**), so the app must manually enforce admission control.
2. **Application Locking:** Record the transaction in a database to prevent concurrent UI mutations.
3. **MDT-Aligned Staging:** Create the staging directory on the same MDT as the destination parent (**LU-17434**) to guarantee the final atomic commit.
4. **Integrity Check:** Compare `st_size` and `st_blocks` to handle sparse files correctly.
5. **Atomic Commit:** `os.rename()` followed by an `os.fsync()` on the parent directory to ensure durability.

---

## 4. Path B: Direct Migration (Big Data / Massive Trees)

When terabytes of data make copying impossible, the Web UI must perform an **In-Place Project Migration**. Because this bypasses ACID safety, the following **Guardrails** are mandatory:

### Guardrail 1: Logical Project Lockdown

The application must implement a "Maintenance Mode" lock on both the source and target project IDs in the database.

* **Reason:** If a user modifies the tree (e.g., adds or deletes files) while the `lfs project` command is walking it, the command may exit with an error, leaving the project accounting in a "split" state.

### Guardrail 2: Hard-Limit Admission Control

Since the `root` user ignores project hard limits, a Direct Migration can accidentally "break" a project by overfilling it.

* **Action:** Calculate the source tree size using a fast metadata scan (`lfs find` or `os.scandir`).
* **Gate:** Refuse to start the migration if `Target_Free_Space < Source_Usage + 10% safety margin`.

### Guardrail 3: Iterative Execution and Verification

Large trees should be processed in batches to manage MDS load and provide progress feedback.

```bash
# Example Administrative Batch Migration
# 1. Set the inheritance flag first (fast)
lfs project -p <NEW_ID> -s /path/to/massive_dir

# 2. Walk and update project IDs
lfs project -p <NEW_ID> -r /path/to/massive_dir

# 3. Final Verification: Check for any 'orphaned' project IDs
lfs find /path/to/massive_dir ! -project <NEW_ID>

```

---

## 5. HSM and DNE Considerations

* **HSM State:** Both paths are blocked by **released** files (on tape). The UI must detect these using `lfs hsm_state` before attempting any move or project change to avoid indefinite I/O hangs.
* **DNE Striping:** For striped directories, the app must ensure that staging or migration logic respects the MDT placement to avoid secondary `EXDEV` errors during final metadata linking.

---

## 6. Stable Conclusions

1. **Copying is for Safety; `lfs project` is for Scale.**
2. **EXDEV is a signal**, not an error. It indicates the boundary of Lustre's native atomicity.
3. **Root Bypass creates Accounting Drift:** Backend processes must manually validate quota limits to prevent projects from becoming unusable for end users.
4. **Application-Level Locking is Mandatory:** Since the filesystem cannot provide a "Transaction Lock" across millions of metadata updates, the Web UI database must manage operation isolation.

---

## References

[1] **POSIX.1-2017** — `rename()` function specification.

[2] **Lustre JIRA: LU-13176** — file-level rename optimization (regular files only).

[3] **Lustre JIRA: LU-17434** — MDS: Support pinning directory to specific MDT.

[4] **Lustre JIRA: LU-17441** — MDC: Separate portal for heavy rename RPCs.

[5] **Lustre JIRA: LU-17426** — MDS parallel cross-directory rename performance.

[6] **Lustre-Discuss Thread 019705** — Confirmation of directory `EXDEV` behavior (Dec 2025).

[7] **Lustre Operations Manual, Section 17.5** — Quota Enforcement and privileged user behaviors.

[8] **LWN.net** — Directory `fsync` requirements for durable rename.

