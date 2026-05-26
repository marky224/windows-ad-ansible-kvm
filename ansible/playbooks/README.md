# Playbooks

Each playbook is named `NN-verb-noun.yml`. The numeric prefix implies execution order when running a full build. `99-` is reserved for verification.

| File | Purpose |
|---|---|
| `00-libvirt-network.yml` | Define + start the `corp-lab` libvirt network |
| `01-provision-dc.yml` | Build the DC VM, unattended Windows install, WinRM bootstrap |
| `02-configure-dc.yml` | Run all `ad_*` roles in order |
| `03-provision-clients.yml` | Build CLIENT01/02 VMs in parallel |
| `04-join-domain.yml` | Domain-join the Windows clients |
| `05-provision-linux.yml` | Build UBUNTU01 from cloud-img + cloud-init |
| `06-join-linux.yml` | realmd / sssd / adcli on UBUNTU01 |
| `99-smoke-test.yml` | End-to-end verification: DNS, AD, DHCP, CA, NTP, login |
| `snapshot.yml` | Take a libvirt snapshot of one or more VMs |
| `rollback.yml` | Restore a VM from a named snapshot |
| `list-snapshots.yml` | Enumerate snapshots |
| `backup-ad.yml` | Run `ops_backup` role against the DC |
| `fire-drill.yml` | Restore latest backup into a sandbox; run smoke test |
| `teardown.yml` | Destroy all VMs (prompts for confirmation) |
| `site.yml` | Orchestrator: 00 → 99 end-to-end |

Playbooks are implemented one at a time alongside the roles they consume. Initial commit (this scaffolding) ships only this README; playbook files arrive with their corresponding roles.
