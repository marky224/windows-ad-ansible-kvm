# ad_sites

Configures AD Sites & Services topology for the Phase 6 multi-site design
(ADR-052), run on the existing DC (**ADDC01**) **before** the Branch replica
(ADDC02) is promoted.

There is no native Ansible module for AD replication sites, so this role uses
inline `ansible.windows.win_powershell` over the ActiveDirectory module
(`New/Set/Get-ADReplicationSite|Subnet|SiteLink`, `Rename-ADObject`), following
the repo's read-state-before-write idempotency + post-write-verify pattern
(ADR-037).

## What it does
1. Renames `Default-First-Site-Name` → `HQ-Site` (keeps ADDC01's membership — no
   server-object move).
2. Creates `Branch-Site` (empty until ADDC02 is promoted into it).
3. Creates the two subnet objects: `10.10.0.0/24` → HQ-Site, `10.20.0.0/24` →
   Branch-Site. ADDC02 (`10.20.0.10`) auto-lands in Branch-Site at promotion via
   this mapping.
4. Creates the `HQ-Branch` site link (both sites, cost 100, 15-min frequency,
   scheduled replication / change-notification **off**, so inter-site behaviour
   is demonstrable).

## Idempotency
Re-runnable; `changed=0` on the second run. Each step reads current state first.

## Scope / safety
Targets the `dc` group only (ADDC01). Pure Configuration-partition changes; **no
control-host surface**. Router-independent (ADDC02 / VyOS not required), so it is
pulled ahead of the VyOS sub-milestone.

## Variables
See `defaults/main.yml`. Site names are templated as PS string literals (not
`win_powershell` parameters) because they contain dashes — see memory
`feedback-win-powershell-args-dash-truncation`.
