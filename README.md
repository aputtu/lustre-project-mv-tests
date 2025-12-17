# Lustre + ZFS Test Environment

This Vagrant environment automates the creation of a Lustre filesystem backed by ZFS on Rocky Linux 9. It is designed to test and demonstrate the behavior of the `mv` command when moving files between directories with different Lustre Project Quota IDs.

## Hypothesis

When moving files between Lustre directories with **different Project IDs**, the kernel returns `EXDEV` (cross-device link error), which forces `mv` to fall back to a slow **copy-and-delete** operation instead of an instant **atomic rename**.

This behavior has significant performance implications when moving large files in environments using Lustre project quotas.

## Overview

The environment is built in two automated phases:

1. **Phase 1 (Kernel):** Installs the Lustre-patched Linux kernel from Whamcloud and configures GRUB to boot into it.
2. **Phase 2 (Storage):** Installs ZFS, configures Lustre (MGS/MDT/OST), sets up test scenarios with different project IDs, and generates test scripts.

### Specifications

| Component | Value |
|-----------|-------|
| OS | Rocky Linux 9 |
| Kernel | Lustre 2.16.x patched (5.14.0) |
| vCPUs | 10 |
| RAM | 20 GB |
| Storage Backend | ZFS (file-backed pools) |
| Network | Private IP `192.168.60.10` |
| Lustre Filesystem | `testfs` mounted at `/mnt/lustre` |

## Prerequisites

- [VirtualBox](https://www.virtualbox.org/) (tested with 7.x)
- [Vagrant](https://www.vagrantup.com/) (tested with 2.4.x)
- **Plugin:** `vagrant-reload` (automatically installed by the Vagrantfile if missing)

### System Requirements

- **Disk:** ~30 GB free space (VM image + ZFS backing files)
- **RAM:** 24+ GB total recommended (20 GB for VM + host overhead)
- **Network:** Internet access for package downloads

## Quick Start

```bash
# 1. Clone or download this directory
cd lustre-zfs-test

# 2. Build the environment (fully automated)
vagrant up

# 3. Connect to the VM
vagrant ssh

# 4. Run the test suite
sudo /root/run_tests.sh
```

The `vagrant up` command will:
1. Create and boot a Rocky Linux 9 VM
2. Install the Lustre-patched kernel (Phase 1)
3. Automatically reboot the VM
4. Configure ZFS and Lustre (Phase 2)
5. Set up test scenarios and generate test scripts

## Test Scenarios

The environment sets up four directories with different project quota configurations:

| Directory | Project ID | Purpose |
|-----------|------------|---------|
| `scenario1_proj100` | 100 | Source for cross-project move |
| `scenario1_proj200` | 200 | Destination (different project) |
| `scenario2_proj300_src` | 300 | Source for same-project move |
| `scenario2_proj300_dst` | 300 | Destination (same project) |

### Test Files

- `testfile_100M.bin` - 100 MB random data (quick tests)
- `testfile_500M.bin` - 500 MB random data (dramatic timing difference)

## Expected Results

### Test 1: EXDEV Detection
Moving a file from `scenario1_proj100` to `scenario1_proj200` (Project 100 → 200):

```
renameat2(...) = -1 EXDEV (Invalid cross-device link)
```

**Result:** `mv` falls back to copy-and-delete.

### Test 2 & 3: Timing Comparison

| File Size | Different Project IDs | Same Project IDs | Speedup |
|-----------|----------------------|------------------|---------|
| 100 MB | ~1-3 seconds | ~0.01 seconds | 100-300x |
| 500 MB | ~3-10 seconds | ~0.01 seconds | 300-1000x |

*Actual times depend on storage performance.*

### Test 4: Workaround
Changing the file's project ID before moving allows atomic rename:

```bash
# Change file's project ID to match destination
lfs project -s -p 200 scenario1_proj100/testfile.bin

# Now move is instant
mv scenario1_proj100/testfile.bin scenario1_proj200/
```

## Available Scripts

| Script | Description |
|--------|-------------|
| `/root/run_tests.sh` | Full test suite with timing comparisons |
| `/root/quick_test.sh` | Quick EXDEV detection check |

## Manual Operations

### Check Project IDs
```bash
# Directory project ID
lfs project -d /mnt/lustre/scenario1_proj100

# File project ID
lfs project /mnt/lustre/scenario1_proj100/testfile_100M.bin

# Recursive listing
lfs project -r /mnt/lustre/
```

### Change Project IDs
```bash
# Set project ID on a file
lfs project -s -p 200 /mnt/lustre/scenario1_proj100/testfile.bin

# Set project ID recursively on directory
lfs project -s -p 100 -r /mnt/lustre/scenario1_proj100/
```

### Lustre Status
```bash
# List NIDs (network identifiers)
lctl list_nids

# Lustre version
lctl version

# Filesystem status
lfs df -h
```

## Troubleshooting

### Vagrant reload fails
If the automatic reboot fails, manually recover:

```bash
vagrant halt
vagrant up
vagrant provision --provision-with lustre_config
```

### Kernel mismatch
Verify the Lustre kernel is running:

```bash
vagrant ssh -c "uname -r"
# Should contain "lustre" (e.g., 5.14.0-503.40.1.el9_5_lustre.x86_64)
```

If not, manually set the default kernel:
```bash
vagrant ssh
sudo grubby --set-default /boot/vmlinuz-*lustre*
sudo reboot
```

### LNet not configured
Check LNet status:
```bash
vagrant ssh
sudo lctl list_nids
# Should show: 192.168.60.10@tcp0
```

If empty, manually configure:
```bash
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if enp0s8  # or eth1
```

### ZFS module not loaded
```bash
vagrant ssh
sudo modprobe zfs
sudo zpool list
```

### Lustre mount issues
```bash
# Check mount status
mount | grep lustre

# Manual mount sequence
sudo mount -t lustre mdtpool/mdt0 /mnt/mdt
sudo mount -t lustre ostpool/ost0 /mnt/ost
sudo mount -t lustre 192.168.60.10@tcp0:/testfs /mnt/lustre
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Rocky Linux 9 VM                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  /mnt/lustre (testfs)                │   │
│  │                    Lustre Client                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│              ┌────────────┴────────────┐                   │
│              │                         │                    │
│  ┌───────────▼───────────┐ ┌──────────▼──────────┐        │
│  │    /mnt/mdt           │ │    /mnt/ost         │        │
│  │   MGS + MDT (mdt0)    │ │    OST (ost0)       │        │
│  │   Metadata Server     │ │   Object Storage    │        │
│  └───────────┬───────────┘ └──────────┬──────────┘        │
│              │                         │                    │
│  ┌───────────▼───────────┐ ┌──────────▼──────────┐        │
│  │   mdtpool (ZFS)       │ │   ostpool (ZFS)     │        │
│  │   /var/lib/lustre-    │ │   /var/lib/lustre-  │        │
│  │      mdt.img (5GB)    │ │      ost.img (15GB) │        │
│  └───────────────────────┘ └─────────────────────┘        │
│                                                             │
│  Network: 192.168.60.10 (tcp0)                             │
└─────────────────────────────────────────────────────────────┘
```

## Cleanup

```bash
# Destroy the VM completely
vagrant destroy -f

# Remove downloaded box (optional)
vagrant box remove rockylinux/9
```

## References

- [Lustre Documentation](https://doc.lustre.org/)
- [Whamcloud Downloads](https://downloads.whamcloud.com/public/lustre/)
- [ZFS on Linux](https://openzfs.github.io/openzfs-docs/)
- [Lustre Project Quotas](https://doc.lustre.org/lustre_manual.xhtml#dbdoclet.projquota)

## License

This test environment is provided as-is for testing and educational purposes.
