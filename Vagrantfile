# Vagrantfile for Lustre + ZFS Test Environment
#
# This environment tests Lustre project quota behavior with mv operations.
# It demonstrates that moving files between directories with different
# Project IDs triggers EXDEV (cross-device link) errors, forcing slow
# copy-and-delete instead of atomic rename.
#
# Usage:
#   vagrant up                 # Full automated setup (Phase 1 + reboot + Phase 2)
#   vagrant ssh                # Connect to VM
#   sudo /root/run_tests.sh    # Run the test suite
#
# Manual recovery if vagrant up hangs after kernel install:
#   vagrant halt && vagrant up && vagrant provision --provision-with lustre_config

# -*- mode: ruby -*-
# vi: set ft=ruby :

### AUTO-INSTALL REQUIRED PLUGIN ###
unless Vagrant.has_plugin?("vagrant-reload")
  system("vagrant plugin install vagrant-reload") || exit!
  puts "Plugin 'vagrant-reload' installed. Restarting command..."
  exec "vagrant " + ARGV.join(" ")
end

Vagrant.configure("2") do |config|
  config.vm.box = "rockylinux/9"
  config.vm.hostname = "lustre-test"

  # Hardware resources - Lustre + ZFS need substantial memory
  config.vm.provider "virtualbox" do |vb|
    vb.name = "lustre-zfs-test"
    vb.memory = "20480"
    vb.cpus = 10
    vb.customize ["modifyvm", :id, "--ioapic", "on"]
    # Improve disk I/O for ZFS operations
    vb.customize ["storagectl", :id, "--name", "SATA Controller", "--hostiocache", "on"]
  end

  # Private network for LNet (Lustre networking)
  config.vm.network "private_network", ip: "192.168.60.10"

  # Disable synced folder to avoid conflicts with Lustre mount
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Copy Ansible playbooks to VM
  config.vm.provision "file", source: "phase1_kernel.yml", destination: "/tmp/phase1_kernel.yml"
  config.vm.provision "file", source: "phase2_lustre.yml", destination: "/tmp/phase2_lustre.yml"

  # === PHASE 1: Install Lustre Kernel ===
  config.vm.provision "kernel_install", type: "ansible_local" do |ansible|
    ansible.playbook = "/tmp/phase1_kernel.yml"
    ansible.install_mode = "default"
    ansible.provisioning_path = "/tmp"
  end

  # === AUTOMATED REBOOT ===
  # Required to boot into the Lustre-patched kernel
  if Vagrant.has_plugin?("vagrant-reload")
    config.vm.provision :reload
  else
    puts "WARNING: vagrant-reload plugin missing. Manual 'vagrant reload' required after Phase 1."
  end

  # === PHASE 2: Configure Lustre + ZFS ===
  config.vm.provision "lustre_config", type: "ansible_local" do |ansible|
    ansible.playbook = "/tmp/phase2_lustre.yml"
    ansible.provisioning_path = "/tmp"
  end
end
