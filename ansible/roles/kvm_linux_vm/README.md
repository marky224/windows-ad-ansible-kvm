# Role: `kvm_linux_vm`

Generic Ubuntu 24.04 VM provisioner. Creates a qcow2 overlay backed by the
cloud image, builds a cloud-init NoCloud seed ISO (user-data + meta-data),
defines the libvirt domain with `virt-install --import`, and waits for SSH.

**Host safety.** Every task runs **unprivileged** on `kvm_host` — `qemu-img`,
`ssh-keygen`, `xorriso`, `virt-install`, `virsh` only, plus files under
`~/vm-lab` and `/tmp`. **No `become`, no mounts, no host-OS edits.** A guard
refuses to run anywhere but `kvm_host`. (See `feedback-never-touch-control-host`.)

**Domain XML profile.**
- Machine type `q35`, **BIOS** firmware (seabios) — cloud images boot cleanly on
  BIOS; avoids NVRAM/Secure-Boot moving parts. UEFI parity with Windows deferred.
- Root disk on `virtio-blk`, qcow2 overlay backed by the cloud image.
- One CD-ROM on SATA: the NoCloud seed ISO (label `CIDATA`).
- NIC on `virtio-net`, MAC-pinned to the `linux_clients` reservation
  (`52:54:00:22:22:01` → DHCP `10.10.0.60`), attached to `corp-lab`.

**SSH key.** A dedicated ed25519 keypair is generated at `~/vm-lab/ssh/corp-lab`
(outside the repo) if absent; the public key is seeded into `labadmin`'s
`authorized_keys`. `group_vars/linux_clients.yml` points the SSH connection at
the private key and at `UserKnownHostsFile=/dev/null` (never writes to the
host's `~/.ssh/known_hosts`).

**Input variables.** `vm_name`, `vm_mac` (required); `linux_ssh_wait_host`
(reserved IP to wait on); `vm_memory_mb`/`vm_vcpus`/`vm_disk_gb`,
`linux_admin_user`, `vm_os_variant`.

**ADRs implemented.** ADR-010 (separate Win/Linux roles), ADR-014 (Ubuntu 24.04),
ADR-023 (unprivileged `qemu:///system`), ADR-024 (storage pools).

**Used by playbook.** `07-provision-linux.yml`.
