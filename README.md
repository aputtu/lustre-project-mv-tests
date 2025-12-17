# README: Lustre + LDISKFS Project Quota Lab

This repository provides a fully automated Vagrant environment to demonstrate the **EXDEV performance penalty** in Lustre filesystems. It proves that moving directories between different Project Quotas triggers a slow "copy-and-delete" operation instead of an instant metadata rename.

## Hypothesis

In Lustre, while files can often be moved across project boundaries atomically, **directories** cannot. When a directory move is attempted across different Project IDs, the kernel returns `EXDEV` (cross-device link error), forcing the `mv` utility to physically copy data and delete the source, resulting in a massive performance drop.

## Prerequisites

Ensure your host machine meets these requirements before starting:
* **VirtualBox**: 7.x recommended.
* **Vagrant**: 2.4.x or newer.
* **Resources**: 24 GB+ RAM (20 GB is allocated to the VM) and 30 GB free disk space.
* **Plugin**: `vagrant-reload` (The `Vagrantfile` will attempt to install this automatically if missing).

## Quick Start

The environment is built in two automated phases (Kernel Install -> Reboot -> Storage Config). You can run the entire experiment with three commands:

```bash
# 1. Build the environment (Fully Automated)
vagrant up

# 2. Connect to the virtual machine
vagrant ssh

# 3. Execute the isolated test suite
sudo /root/run_tests.sh

```

## Test Scenarios

The suite runs four isolated tests to compare performance:

| Test | Scenario | Expected Logic | Result |
| --- | --- | --- | --- |
| **Test 0** | File Move (Project 100 → 200) | Atomic Rename | Baseline Success |
| **Test 1** | Directory Move (Project 100 → 200) | **EXDEV Error** | Copy+Delete Fallback |
| **Test 2** | 500MB Move (Different Projects) | Copy + Delete | **Slow Path** (~100x slower) |
| **Test 3** | 500MB Move (Same Project) | Atomic Rename | **Fast Path** (Instant) |
| **Test 4** | Workaround (Align ID first) | Atomic Rename | Fast Path Restored |

## Technical Overview

* **OS**: Rocky Linux 9.
* **Kernel**: Lustre-patched 5.14.0.
* **Lustre Version**: 2.16.0.
* **Storage**: LDISKFS-backed MGS, MDT, and OST using loopback devices.
* **Network**: LNet configured on `192.168.60.10@tcp0`.

## Troubleshooting

* **Vagrant hangs after Kernel Install**: If the auto-reboot fails, run `vagrant halt`, then `vagrant up`, followed by `vagrant provision --provision-with lustre_config`.
* **Idempotency**: The Phase 2 playbook includes safety checks. If you need to re-run it, it will skip formatting if the Lustre filesystem is already live.
* **Manual Verification**: You can manually check a directory's Project ID using:
`lfs project -d /mnt/lustre/scenario1_proj100`.

## Cleanup

To completely remove the environment and free up host resources:

```bash
vagrant destroy -f

```

## Copyleft License
This test environment is provided for educational and research purposes regarding distributed filesystem behavior.

