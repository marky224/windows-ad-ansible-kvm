# Playbooks

Each playbook is named `NN-verb-noun.yml`; the numeric prefix is the execution order for a full build (`99-` is verification). The `00→08` chain plus `99` builds the base lab end-to-end; `09→17` are the **Phase 6** (multi-site) build steps; the rest are operational utilities.

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
| `09-configure-sites.yml` | **Phase 6** (ADR-052): AD Sites & Services on ADDC01 — `Default-First-Site-Name`→`HQ-Site`, `Branch-Site`, subnet objects, `HQ-Branch` link (`ad_sites`); precedes the Branch replica |
| `10-provision-addc02.yml` | **Phase 6**: build the `ADDC02-corp` Branch replica VM on the isolated `corp-branch` net (10.20.0.10), DNS→ADDC01 pre-promo (`kvm_windows_vm`) |
| `11-provision-vyos.yml` | **Phase 6** (ADR-052/054): build the `VYOS01` inter-site router VM from the VyOS rolling ISO (`net_router_vyos`); boots to the live installer for a one-time `install image` |
| `12-configure-vyos.yml` | **Phase 6**: configure VYOS01 over `vyos.vyos` (interfaces, static routes 10.10⇄10.20, ~40 ms netem WAN sim) + wire the HQ→Branch route on ADDC01 |
| `13-promote-addc02.yml` | **Phase 6** (ADR-052/055): promote ADDC02 → writable Branch-Site replica DC + GC (`ad_dc_replica`); **one-shot** across the identity switch; verifies replication both directions |
| `14-dns-crosspoint.yml` | **Phase 6**: set each DC's resolver self-first (loopback preferred, cross-site partner alternate); verify no DNS island + replication clean both ways |
| `15-branch-dns-forwarders.yml` | **Phase 6** (ADR-055(c)): ADDC02 forward-only DNS → ADDC01 (hub) + root-hints off (the isolated branch has no internet path) |
| `16-branch-dhcp.yml` | **Phase 6** (ADR-056): Branch DHCP scope `10.20.0.0/24` on ADDC02 (`ad_dhcp` via `group_vars/dc_replica.yml` overrides) |
| `17-dhcp-failover.yml` | **Phase 6** (ADR-056): reciprocal hot-standby DHCP failover — preflight gate → HQ create on ADDC01 → Branch create on ADDC02 → poll to `State=Normal`; `become: runas` for the partner second-hop |
| `99-smoke-test.yml` | End-to-end verification: DNS, AD, DHCP, CA, NTP, login (shared DC checks in `tasks/smoke-dc.yml`) |
| `snapshot.yml` | Take a disk-level snapshot of one or more VMs (`ops_snapshot`) |
| `rollback.yml` | Restore a VM from a named snapshot (`ops_snapshot`) |
| `list-snapshots.yml` | Enumerate snapshots (`ops_snapshot`) |
| `backup-ad.yml` | AD state backup to the dedicated drive (`ops_backup`, ADR-046) |
| `fire-drill.yml` | Restore a snapshot/backup into an isolated sandbox, smoke-test, always tear down (`ops_firedrill`, ADR-047) |
| `teardown.yml` | Destroy `*-corp` VMs + their live disks; preview unless `-e confirm=DESTROY`; preserves `/mnt/dc-backups` + `.<label>` snapshots (`ops_teardown`, ADR-048) |
| `site.yml` | Orchestrator for the **base lab**: static-imports `00 → 08` + `99-smoke-test` (NOT the Phase 6 `09→17` — those have interactive + one-shot steps); fail-fast preflight + phase tags (`network`/`dc`/`services`/`advanced`/`clients`/`linux`/`smoke`) (ADR-049/050) |

Shared task files live in `tasks/` — e.g. `smoke-dc.yml`, the "healthy DC" checks reused by both `99-smoke-test.yml` and `fire-drill.yml`.
