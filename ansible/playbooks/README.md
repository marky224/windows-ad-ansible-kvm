# Playbooks

Each playbook is named `NN-verb-noun.yml`; the numeric prefix is the execution order for a full build (`99-` is verification). The `00→08` chain plus `99` builds the lab end-to-end; the rest are operational utilities.

| File | Purpose |
|---|---|
| `00-libvirt-network.yml` | Define + start the `corp-lab` libvirt network |
| `slipstream-iso.yml` | `kvm_iso_slipstream`: DISM-patch the Server 2025 install ISO (M3.5, ADR-035) |
| `01-provision-dc.yml` | Build the DC VM, unattended Windows install, WinRM bootstrap |
| `02-configure-dc.yml` | `ad_dc` (forest promotion) → `ad_admins` (named domain admin) → `ad_harden_builtin_admin` (RID 500) |
| `03-configure-services.yml` | DC-resident services: `ad_dns` → `ad_dhcp` → `ad_ntp` |
| `04-configure-services-advanced.yml` | Advanced services: `ad_cs` → `ad_gpo` → `ad_wsus` |
| `05-provision-clients.yml` | Build CLIENT01/02 Win 11 VMs in parallel |
| `06-join-domain.yml` | Domain-join the Windows clients (`domain_join_windows`) |
| `07-provision-linux.yml` | Build UBUNTU01 from cloud-img + cloud-init |
| `08-join-linux.yml` | realmd / sssd / adcli on UBUNTU01 (`domain_join_linux`) |
| `99-smoke-test.yml` | End-to-end verification: DNS, AD, DHCP, CA, NTP, login (shared DC checks in `tasks/smoke-dc.yml`) |
| `snapshot.yml` | Take a disk-level snapshot of one or more VMs (`ops_snapshot`) |
| `rollback.yml` | Restore a VM from a named snapshot (`ops_snapshot`) |
| `list-snapshots.yml` | Enumerate snapshots (`ops_snapshot`) |
| `backup-ad.yml` | AD state backup to the dedicated drive (`ops_backup`, ADR-046) |
| `fire-drill.yml` | Restore a snapshot/backup into an isolated sandbox, smoke-test, always tear down (`ops_firedrill`, ADR-047) |
| `teardown.yml` | Destroy `*-corp` VMs + their live disks; preview unless `-e confirm=DESTROY`; preserves `/mnt/dc-backups` + `.<label>` snapshots (`ops_teardown`, ADR-048) |
| `site.yml` | Orchestrator: static-imports `00 → 99` end-to-end; fail-fast preflight + phase tags (`network`/`dc`/`services`/`advanced`/`clients`/`linux`/`smoke`) + `99-smoke-test` as the final gate (ADR-049) |

Shared task files live in `tasks/` — e.g. `smoke-dc.yml`, the "healthy DC" checks reused by both `99-smoke-test.yml` and `fire-drill.yml`.
