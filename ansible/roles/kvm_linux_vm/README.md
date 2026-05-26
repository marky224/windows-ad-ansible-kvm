# Role: `kvm_linux_vm`

Generic Linux VM provisioner. Clones the Ubuntu 24.04 cloud-img as a qcow2 backing disk, builds a cloud-init NoCloud seed ISO (user-data + meta-data), defines the libvirt domain, boots.

**Domain XML profile.**
- Machine type `q35`, BIOS or UEFI (UEFI for parity with Windows).
- Disk on `virtio-blk`, qcow2 backed by the cloud-img.
- One CD-ROM on SATA: cloud-init NoCloud seed ISO.
- NIC on `virtio-net`, attached to `corp-lab` network.

**Input variables.**
- `vm_name`, `vm_memory_mb`, `vm_vcpus`, `vm_disk_gb`
- `cloud_init_user_data_template`, `cloud_init_meta_data_template`
- Initial user/password from `vault_ubuntu_initial_password`.

**ADRs implemented.** ADR-010 (separate Win/Linux roles), ADR-014 (Ubuntu 24.04), ADR-024.

**Used by playbook.** `05-provision-linux.yml`.
