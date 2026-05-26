# windows-ad-ansible-kvm

**Ansible Infrastructure-as-Code for a production-quality MSP-style Active Directory lab on KVM/libvirt.**

End-to-end automated build: from a bare Ubuntu 24.04 host, Ansible provisions a Windows Server 2025 Domain Controller (with AD DS, DNS, DHCP, AD CS, NTP, WSUS), two Windows 11 Enterprise clients, and an Ubuntu 24.04 member server — all joined to a single forest, on an isolated `10.10.0.0/24` network.

> **Predecessor:** [marky224/Active-Directory-Domain-Controller-Provisioning](https://github.com/marky224/Active-Directory-Domain-Controller-Provisioning) — the original PowerShell-imperative version. Frozen, not under active development. This repo replaces it with an IaC-first Ansible approach.

---

## What you get

| VM | OS | Role |
|---|---|---|
| `ADDC01-corp` | Windows Server 2025 Std + Desktop Experience | Domain Controller — AD DS, DNS, DHCP, AD CS (Enterprise Root CA), NTP, WSUS |
| `CLIENT01-corp` | Windows 11 Enterprise | Domain-joined workstation |
| `CLIENT02-corp` | Windows 11 Enterprise | Domain-joined workstation |
| `UBUNTU01-corp` | Ubuntu 24.04 LTS Server | Domain-joined Linux server (realmd + sssd) |

- **Forest:** `corp.markandrewmarquez.com` (NetBIOS: `CORP`)
- **Subnet:** `10.10.0.0/24` — DC owns DHCP (scope `.100-.199`) with MAC-tied reservations for clients
- **GPO baseline:** Microsoft Security Compliance Toolkit Server 2025 baseline + 12 lab-specific overrides
- **AD state backup:** nightly `wbadmin systemstatebackup` to a host-side Samba share + `Backup-GPO`/`Backup-CARoleService`/`Export-DhcpServer` exports via WinRM `fetch`
- **Snapshots:** automatic at each provisioning phase (`vm-built`, `ad-promoted`, `roles-installed`, `clients-joined`, `linux-joined`)
- **Fire drill:** quarterly playbook restores the latest backup to a sandbox VM on an isolated network and runs the smoke test against it

Total wall-clock for a clean provision: **~60–75 minutes**, mostly unattended.

---

## Hardware requirements

- x86_64 with VT-x or AMD-V enabled in BIOS
- 20 GB free RAM (32 GB recommended)
- 250 GB free disk (400 GB recommended)
- Ubuntu 24.04 LTS host (other Debian-family distros should work; package names may differ)

---

## Quickstart

> Full step-by-step in the [Prerequisites](docs/PREREQUISITES.md) and [Runbook](docs/RUNBOOK.md). The summary below is the happy path.

### 1. Install host packages

```bash
sudo apt update && sudo apt install -y \
  qemu-kvm libvirt-daemon-system libvirt-clients virtinst bridge-utils \
  xorriso ansible-core \
  python3-libvirt python3-lxml python3-winrm python3-pip \
  swtpm swtpm-tools ovmf \
  virt-manager virt-viewer \
  samba samba-common-bin libnss-libvirt wimtools pipx

sudo usermod -aG libvirt,kvm "$USER"
sudo systemctl enable --now libvirtd
# Log out and back in for group changes to take effect.
```

### 2. Get ISOs

```bash
mkdir -p /home/$USER/vm-lab/{disks,iso,seed-iso,backups,snapshots}
cd /home/$USER/vm-lab/iso

# virtio-win 0.1.271 (PINNED — do NOT use 0.1.285)
curl -sSL -O https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.271-1/virtio-win-0.1.271.iso
ln -sf virtio-win-0.1.271.iso virtio-win.iso

# Ubuntu 24.04 cloud image
curl -sSL -O https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

Plus, downloaded manually through Microsoft eval forms (no credit card, no product key):
- **Windows Server 2025**: https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025 → save as `WindowsServer2025.iso`
- **Windows 11 Enterprise**: https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise → save as `Windows11Enterprise.iso`

### 3. Vault password + secrets

```bash
openssl rand -base64 32 > ~/.ansible-vault-pass-corp-lab
chmod 600 ~/.ansible-vault-pass-corp-lab
# Back up the contents to your password manager NOW. If you lose it, the vault is unrecoverable.
```

### 4. Install Ansible collections

```bash
ansible-galaxy collection install -r ansible/requirements.yml
```

### 5. Set secrets

```bash
ansible-vault edit ansible/group_vars/all/vault.yml
# Set: vault_local_admin_password, vault_dsrm_password, vault_domain_admin_password,
#      vault_ca_passphrase, vault_dcbackup_smb_password
```

### 6. Run the lab

```bash
cd ansible
ansible-playbook playbooks/00-libvirt-network.yml
ansible-playbook playbooks/01-provision-dc.yml
ansible-playbook playbooks/02-configure-dc.yml
ansible-playbook playbooks/03-provision-clients.yml
ansible-playbook playbooks/04-join-domain.yml
ansible-playbook playbooks/05-provision-linux.yml
ansible-playbook playbooks/06-join-linux.yml
ansible-playbook playbooks/99-smoke-test.yml
```

Or all in one:
```bash
ansible-playbook playbooks/site.yml
```

---

## Architecture

- **Single mental model:** Ansible roles + playbooks. No standalone PowerShell scripts. Where Windows config has no native Ansible module (DNS server, DHCP, WSUS, GPO), the role uses inline `ansible.windows.win_powershell` blocks within YAML tasks.
- **Hypervisor:** KVM/libvirt with `community.libvirt`. Each VM defined via libvirt XML rendered from Jinja2.
- **Windows install:** thin `Autounattend.xml` (per-VM seed ISO via `xorriso`) gets the OS unattended + WinRM listening. All AD configuration owned by Ansible roles thereafter.
- **Linux install:** cloud-init NoCloud datasource (also per-VM seed ISO).
- **Authentication:** `ansible-vault` from day one — vault password file at `~/.ansible-vault-pass-corp-lab` (never committed).

## Roles

| Role | Purpose |
|---|---|
| `kvm_network` | Define + start `corp-lab` libvirt network (10.10.0.0/24, NAT, no DHCP) |
| `kvm_windows_vm` | Generic Windows VM provisioning (Autounattend seed, libvirt domain, boot, wait for WinRM) |
| `kvm_linux_vm` | Generic Linux VM provisioning (cloud-init seed, boot, wait for SSH) |
| `ad_dc` | Static IP, AD DS install, forest creation (`microsoft.ad.domain`) |
| `ad_dns` | DNS forwarders, reverse zone |
| `ad_dhcp` | DHCP scope + MAC-tied reservations for known clients |
| `ad_cs` | Enterprise Root CA install + `cs_authority` + `cs_template` |
| `ad_gpo` | Import MSFT SCT Server 2025 baseline + 12 lab-specific overrides |
| `ad_ntp` | PDC NTP authority configuration |
| `ad_wsus` | WSUS install + async sync trigger |
| `domain_join_windows` | Win 11 client domain join (`microsoft.ad.membership`) |
| `domain_join_linux` | Ubuntu domain join via `realmd` + `sssd` |
| `ops_backup` | AD state backup orchestration (SMB to host + WinRM `fetch`) |

## Playbooks

`00-libvirt-network.yml`, `01-provision-dc.yml`, `02-configure-dc.yml`, `03-provision-clients.yml`, `04-join-domain.yml`, `05-provision-linux.yml`, `06-join-linux.yml`, `99-smoke-test.yml`, plus operational utilities: `snapshot.yml`, `rollback.yml`, `list-snapshots.yml`, `backup-ad.yml`, `fire-drill.yml`, `teardown.yml`.

---

## Project status

🚧 **Under Construction**

This README will be updated as implementation lands. For up-to-date design rationale and architectural decisions, see the `docs/` directory in this repository.

---

## License

MIT. See [LICENSE](LICENSE).
