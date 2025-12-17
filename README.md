# Lustre Project Quota Lab

This repository provides a fully automated Vagrant environment demonstrating **project quota behavior** in Lustre filesystems, specifically:

1. **EXDEV Performance Penalty** — Directory moves across project boundaries trigger slow copy-and-delete
2. **Double Billing Problem** — Quota usage temporarily doubles during cross-project moves
3. **LU-13176 Behavior** — Lustre 2.15+ allows atomic file renames but NOT directory renames

## The Problem

When users move directories between different project quotas, two issues occur:

### Performance Impact
```
Same project move:     ~0.01 seconds (atomic rename)
Cross-project move:    ~60 seconds for 500MB (copy + delete)
                       ≈ 6000x slower
```

### Billing Impact
```
Before move:   Project A = 500MB, Project B = 0MB    → Billed: 500MB
During move:   Project A = 500MB, Project B = 500MB  → Billed: 1000MB ⚠️
After move:    Project A = 0MB,   Project B = 500MB  → Billed: 500MB
```

If billing snapshots occur during the move window, users are **double-charged**.

## Prerequisites

| Requirement | Minimum |
|-------------|---------|
| VirtualBox  | 7.x     |
| Vagrant     | 2.4.x   |
| RAM         | 24 GB (20 GB allocated to VM) |
| Disk        | 30 GB free |

The `vagrant-reload` plugin is installed automatically.

## Quick Start

```bash
# Build environment (Phase 1: Kernel → Reboot → Phase 2: Lustre)
vagrant up

# Connect to VM
vagrant ssh

# Run test suite
sudo /root/run_tests.sh

# Optional: Run with larger test data
TEST_SIZE_MB=1000 sudo /root/run_tests.sh
```

## Test Scenarios

| Test | Operation | Expected Behavior | Quota Impact |
|------|-----------|-------------------|--------------|
| **1** | File move (Proj A→B) | Atomic rename (LU-13176) | Instant transfer |
| **2** | Directory move (Proj A→B) | EXDEV → copy+delete | **Double billing window** |
| **3** | Directory move (same project) | Atomic rename | No change |
| **4** | Workaround: `lfs project` first | Atomic after ID change | No double billing |

## Understanding the Results

### Test 2 Output (The Problem)
```
[BEFORE]
  Project 100: 500 MB
  Project 200: 0 KB
  Combined:    500 MB

Quota samples collected: 12
Peak combined usage: 987.5 MB
⚠ DOUBLE BILLING DETECTED!
  Peak was 487.5 MB above baseline

[AFTER]
  Project 100: 0 KB
  Project 200: 500 MB
  Combined:    500 MB
```

### Test 4 Output (The Solution)
```
Step 1: lfs project -s -p 200 -r /path/to/dir
  Re-project time: 0.02s

Step 2: mv /src/dir /dst/dir
  Move time: 0.01s

Advantage: No double-billing! Quota transfer is atomic.
```

## Workaround for Production

Instead of:
```bash
mv /project_a/bigdir /project_b/bigdir  # SLOW + DOUBLE BILLING
```

Do this:
```bash
# Step 1: Change project ID in place (metadata only, fast)
lfs project -s -p <NEW_PROJECT_ID> -r /project_a/bigdir

# Step 2: Move directory (now same project, atomic)
mv /project_a/bigdir /project_b/bigdir
```

## Technical Details

| Component | Value |
|-----------|-------|
| OS | Rocky Linux 9 |
| Kernel | Lustre-patched 5.14.0 |
| Lustre | 2.16.0 |
| Backend | LDISKFS (ext4 with Lustre patches) |
| Storage | Loopback devices (MDT: 2GB, OST: 10GB) |
| Network | LNet on 192.168.60.10@tcp0 |

### Key Reference

**LU-13176**: *Enable rename of regular file across projid directories without copy*
- Landed in Lustre 2.15
- Allows **files** to be renamed atomically across project IDs
- **Directories** still return EXDEV (by design)
- [Jira Ticket](https://jira.whamcloud.com/browse/LU-13176)

## File Structure

```
.
├── Vagrantfile           # VM configuration with two-phase provisioning
├── phase1_kernel.yml     # Installs Lustre-patched kernel
├── phase2_lustre.yml     # Configures Lustre + creates test script
└── README.md             # This file
```

## Troubleshooting

### Vagrant hangs after kernel install
```bash
vagrant halt
vagrant up
vagrant provision --provision-with lustre_config
```

### Check project IDs manually
```bash
lfs project -d /mnt/lustre/scenario1_proj100
lfs project -r /mnt/lustre/scenario1_proj100/
```

### View quota usage
```bash
lfs quota -p 100 /mnt/lustre
lfs quota -p 200 /mnt/lustre
```

### Re-run tests without rebuild
```bash
vagrant ssh
sudo /root/run_tests.sh
```

## Cleanup

```bash
vagrant destroy -f
```

## Recommendations for HPC Administrators

1. **Documentation**: Warn users that cross-project directory moves are slow
2. **Billing**: Avoid snapshot timing during known migration windows
3. **Training**: Teach users the `lfs project` workaround
4. **Monitoring**: Alert on sustained high quota delta between projects
5. **Automation**: Provide wrapper scripts that do the workaround automatically

## Copyleft License

This test environment is provided for educational and research purposes.
