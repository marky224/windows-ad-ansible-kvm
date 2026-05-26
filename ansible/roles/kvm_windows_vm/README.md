# Role: `kvm_windows_vm`

Generic Windows VM provisioner. Renders `Autounattend.xml` (with inline WinRM bootstrap per ADR-021), builds a per-VM seed ISO via `xorriso`, defines the libvirt domain, boots, and waits for WinRM HTTPS:5986 to come up.

**Domain XML profile (ADR-020).**
- Machine type `q35`, OVMF UEFI (Secure-Boot variant for Server 2025 / Win 11).
- TPM 2.0 emulated via `swtpm`.
- Boot disk on `virtio-scsi` (`vioscsi`) controller, qcow2, `discard=unmap`.
- Three CD-ROMs on SATA: Windows install media, virtio-win 0.1.271 (ADR-018), per-VM seed ISO.
- NIC on `virtio-net`, attached to `corp-lab` network.
- Boot order: cdrom first, disk second.

**Input variables (key set; defaults in `defaults/main.yml` once implemented).**
- `vm_name`, `vm_memory_mb`, `vm_vcpus`, `vm_disk_gb`
- `install_iso_path`, `wim_index`, `os_variant` (e.g. `win2k25`)
- `autounattend_template` (Jinja2 template path)
- `static_ip` (DC only; clients use DHCP from the DC)

**ADRs implemented.** ADR-018, ADR-020, ADR-021, ADR-024.

**Used by playbooks.** `01-provision-dc.yml`, `03-provision-clients.yml`.
