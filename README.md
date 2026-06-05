# windows-ad-ansible-kvm

**Ansible Infrastructure-as-Code for a production-quality, two-site Active Directory environment on KVM/libvirt — provisioned from bare ISOs and verified end to end.**

Active Directory is still the identity backbone of most enterprise and MSP environments, and what
separates operators from engineers is designing it for **resilience**, automating it so it's
**reproducible**, and **proving it recovers** when a controller fails. This project delivers a complete
two-site AD forest as Ansible Infrastructure-as-Code on a single Linux KVM/libvirt host — domain
controllers, DNS, DHCP, AD CS, WSUS, GPO baselines, Windows and Linux domain members, and a second
**isolated** branch site with cross-site replication and DHCP failover — provisioned from bare install
media and verified end to end. The control host is never modified by the automation — a hard safety
boundary enforced in the roles themselves.

---

## Architecture

### Component walkthrough

**Born unattended.** Every machine starts from a per-VM install ISO — a Windows `Autounattend.xml` or a
Linux cloud-init seed — that boots, installs, and brings up WinRM or SSH with no one at the console. The
`kvm_windows_vm` and `kvm_linux_vm` roles define each libvirt domain (q35 + UEFI Secure Boot + TPM 2.0),
attach the media, and wait for the management channel to answer — about twelve minutes from cold boot to
a working desktop.

![Unattended Windows Server 2025 install begins](docs/screenshots/milestone2-01-setup-launching.png)
![Install proceeds without intervention](docs/screenshots/milestone2-03-installing-83pct.png)
![OOBE auto-skipped to the desktop, WinRM over HTTPS listening](docs/screenshots/milestone2-04-desktop-after-autologon.png)

**Patched before it ever runs.** The Server 2025 media is DISM-slipstreamed (`kvm_iso_slipstream`) so the
controller boots already at build `26100.32860` — current on day one, with no post-install patch window.

![Patched Server 2025 build, slipstreamed at install time](docs/screenshots/milestone-3.5-patched-pre-promotion.png)

**One controller, the services a real site needs.** `ad_dc` promotes ADDC01 into the forest
`corp.markandrewmarquez.com` — all five FSMO roles and a Global Catalog — then a named-admin role and
RID-500 hardening lock down the built-in Administrator. Focused roles layer the services on top:
AD-integrated DNS (`ad_dns`), DHCP with reservations (`ad_dhcp`), an Enterprise Root CA (`ad_cs`), a
Microsoft Security Compliance Toolkit GPO baseline (`ad_gpo`), authoritative time (`ad_ntp`), and WSUS
(`ad_wsus`).

![Forest live — ADDC01 holds all FSMO roles, named admin established](docs/screenshots/milestone-3-dc-promoted.png)

**Endpoints that actually join.** Two Windows 11 Enterprise workstations join the domain with a real
vTPM 2.0 and pick up a machine certificate by **autoenrollment** from the CA (`domain_join_windows`) —
GPO-driven, with no manual request.

![CLIENT01 — Windows 11 Enterprise, domain-joined and autoenrolled](docs/screenshots/milestone-6-client01-win11-desktop.png)

The *same* forest serves Linux: an Ubuntu 24.04 server joins through `realmd`/`sssd` (`domain_join_linux`),
resolving AD identities with `id` and honoring Domain Admins for `sudo`.

![UBUNTU01 — Ubuntu 24.04 domain-joined via realmd/sssd](docs/screenshots/milestone-6-ubuntu01-domain-joined.png)

**A second site — not a second subnet on paper.** A genuinely isolated branch sits on its own
`10.20.0.0/24` network, reachable from HQ only through a VyOS router (`net_router_vyos`) that shapes a
~40 ms WAN. `ad_dc_replica` promotes ADDC02 as a replica DC + Global Catalog; `ad_sites` lays down AD
Sites & Services with a costed, scheduled site link; and self-first DNS plus reciprocal hot-standby DHCP
failover let the branch ride out a WAN cut — or the loss of HQ entirely, which is what the
[disaster-recovery drills](#disaster-recovery) below put to the test.

All of it — both sites, every role — is built from bare install media in about 60–75 minutes,
idempotently (two-run gates), mostly unattended, and gated end to end by a smoke test. The full topology:

```mermaid
flowchart TB
    INET([Internet])
    HOST["KVM/libvirt host - Ansible control node<br/>HQ NAT gateway 10.10.0.1 - branch mgmt leg 10.20.0.2"]
    INET --- HOST

    subgraph HQ["HQ-Site - 10.10.0.0/24"]
        ADDC01["ADDC01 - 10.10.0.10<br/>DC - 5 FSMO - GC<br/>DNS - DHCP - AD CS - WSUS - NTP"]
        C1["CLIENT01 - .50<br/>Windows 11 Enterprise"]
        C2["CLIENT02 - .51<br/>Windows 11 Enterprise"]
        U1["UBUNTU01 - .60<br/>Ubuntu 24.04 - realmd/sssd"]
    end

    subgraph BR["Branch-Site - 10.20.0.0/24 - isolated"]
        ADDC02["ADDC02 - 10.20.0.10<br/>Replica DC - GC<br/>DNS forward-to-hub - DHCP standby"]
    end

    VYOS["VyOS router<br/>eth0 10.10.0.2 - eth1 10.20.0.1<br/>~40 ms WAN - DHCP relay"]

    HOST --- ADDC01
    ADDC01 --- VYOS
    VYOS --- ADDC02
    ADDC01 -.->|AD replication, 15-min schedule| ADDC02
    ADDC01 -.->|hot-standby DHCP failover| ADDC02
    C1 -.->|cross-site DNS and auth failover route| ADDC02
```

---

## Disaster recovery

Multi-site redundancy is only real if it survives the failure it's designed for — so that failure is
**rehearsed**, not assumed. Two complementary drills, backed by a documented FSMO-seizure runbook:

- a **live, non-destructive failover drill** that gracefully powers off the live HQ DC and proves the
  branch DC carries authentication, DNS, and DHCP, then recovers and re-converges, and
- an **isolated-sandbox FSMO-seize rehearsal** that clones the branch DC into an isolated network and
  *actually* seizes all five FSMO roles — the only safe way to execute a real seizure, since a
  seized-from DC must never rejoin the domain.

The sequence below is the live failover drill, end to end:

```mermaid
sequenceDiagram
    autonumber
    participant C1 as CLIENT01 HQ
    participant A1 as ADDC01 HQ FSMO
    participant V as VyOS WAN
    participant A2 as ADDC02 Branch GC
    Note over A1: ADDC01 outage - graceful power-off
    C1->>V: DNS and Kerberos auth via the option-121 branch route
    V->>A2: forwarded across the inter-site link
    A2-->>C1: resolves the forest and authenticates via the GC
    Note over A2: declares DHCP PARTNER DOWN, serves a relayed lease
    Note over A1,A2: ADDC01 returns
    A1->>A2: replication re-converges in both directions
    Note over A1,A2: DHCP failover returns to State=Normal
```

Cross-site failover depends on HQ clients being able to *reach* the branch DC across the link. Because
the branch network is isolated and HQ clients' gateway is the host, the branch route is delivered to
every HQ client centrally via **DHCP option 121** (classless static routes) — chosen over the legacy
option 249 for Windows 11 24H2 option-type safety, with the default route encoded per RFC 3442 so
clients keep internet, and replicated to the standby DHCP server. The result below is CLIENT01 with the
branch route installed — reaching the branch DC across the inter-site link and resolving the forest via
it: the path that carries DNS and authentication when the HQ DC is offline.

![CLIENT01 reaching the branch DC across the inter-site link and resolving the forest via it — the cross-site DNS/auth failover path](docs/assets/cross-site-failover-client01.png)

---

## What this demonstrates

**Active Directory, in depth.** Multi-site design (Sites/Subnets/site-links, with the replication
schedule honored — proven by an object round-trip, not assumed), replication topology, FSMO + Global
Catalog, AD-integrated DNS, AD CS with machine-certificate autoenrollment, GPO baselines (Microsoft
Security Compliance Toolkit), and WSUS.

**Networking.** Subnetting and an explicit IP plan, inter-site routing on VyOS, WAN-latency simulation
(`netem`), and DHCP end to end — scopes, exclusions, MAC reservations, **hot-standby failover**,
cross-site **relay**, and **classless static routes (option 121)** — alongside self-first / local-first
DNS resilience for a WAN-separated, one-DC-per-site topology.

**IaC & automation.** ~25 Ansible roles and playbooks with strict idempotency (two-run gates), Windows
configuration via inline `win_powershell` where no native module exists, `ansible-vault` from day one, a
`site.yml` orchestrator with fail-fast (`any_errors_fatal`), and CI guardrails (`ansible-lint` +
`gitleaks`).

**Virtualization.** KVM/libvirt with q35 / UEFI Secure Boot / TPM 2.0, unattended Windows Server 2025
and Windows 11 installs from slipstreamed media, and snapshot / backup / fire-drill / teardown tooling.

---

## Project structure

```
windows-ad-ansible-kvm/
├── ansible/
│   ├── ansible.cfg                 # inventory, vault, and connection defaults
│   ├── requirements.yml            # collections: microsoft.ad, ansible.windows, community.{windows,libvirt}, vyos.vyos, ...
│   ├── inventory/
│   │   ├── lab.yml                 # the fleet
│   │   └── group_vars/             # all/ (+ vault), dc, dc_replica, clients, linux_clients, routers, windows
│   ├── playbooks/                  # NN-verb-noun.yml, ordered for full builds
│   │   ├── site.yml                # one-command orchestrator (00 -> 99) with fail-fast
│   │   ├── 00..08-*.yml            # base build: network -> DC -> services -> clients -> linux -> join
│   │   ├── 09..18-*.yml            # multi-site: AD sites, ADDC02 replica, VyOS, DNS cross-point, branch DHCP + failover + relay
│   │   ├── verify-multisite*.yml   # read-only two-site health + schedule round-trip proof
│   │   ├── dr-failover-drill.yml   # live, non-destructive DR drill
│   │   ├── snapshot.yml · backup-ad.yml · fire-drill.yml · teardown.yml   # operations
│   │   └── 99-smoke-test.yml       # end-to-end verification gate
│   ├── roles/                      # 23 roles — see Roles below
│   └── files/                      # templates, GPO baseline backups, seed assets
├── docs/
│   ├── screenshots/                # provisioning milestones (shown above)
│   └── assets/                     # architecture + cross-site failover proof images
├── .github/                        # CI: gitleaks secret scan
├── .pre-commit-config.yaml         # gitleaks pre-commit hook
├── .gitleaks.toml
├── LICENSE
└── README.md
```

---

## Roles

| Role | What it does |
|---|---|
| `kvm_network` | Define the libvirt networks — NAT HQ + isolated branch |
| `kvm_windows_vm` | Provision a Windows VM from a custom install ISO; bootstrap WinRM/HTTPS |
| `kvm_linux_vm` | Provision a Linux VM from a cloud image + cloud-init seed |
| `kvm_iso_slipstream` | DISM-slipstream cumulative updates into the Server 2025 ISO |
| `ad_dc` | Install AD DS and create the forest; verify with `dcdiag` |
| `ad_admins` | Create the named domain admin (`madmin-da`) |
| `ad_harden_builtin_admin` | RID-500 hardening of the built-in Administrator |
| `ad_dns` | Forwarders, reverse zone, record scavenging |
| `ad_dhcp` | Scope, exclusions, MAC reservations, options; cross-site failover + relay + option 121 |
| `ad_ntp` | PDC emulator as the authoritative time source |
| `ad_cs` | Enterprise Root CA, templates, machine autoenrollment |
| `ad_gpo` | Microsoft SCT baseline + delta + autoenrollment GPOs |
| `ad_wsus` | WSUS install, product/classification selection, sync |
| `ad_sites` | AD Sites & Services — sites, subnets, costed/scheduled link |
| `ad_dc_replica` | Promote ADDC02 as a Branch replica DC + Global Catalog |
| `net_router_vyos` | VyOS inter-site routing, DHCP relay, ~40 ms `netem` WAN |
| `domain_join_windows` | Join Windows 11 clients to the domain |
| `domain_join_linux` | Join Ubuntu via `realmd`/`sssd` |
| `ops_backup` | AD system-state backup (`wbadmin`) + config exports |
| `ops_snapshot` | Disk snapshot / list / rollback |
| `ops_firedrill` | Restore into an isolated sandbox, verify, always tear down |
| `ops_teardown` | Destroy the fleet's VMs + disks (guarded, preview by default) |
| `_common` | Shared defaults and handlers |

---

## License

Proprietary — all rights reserved. See [LICENSE](LICENSE). Source-available for review; no use, copying, modification, or redistribution without prior written permission.
