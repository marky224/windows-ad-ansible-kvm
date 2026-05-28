# Role: `ad_dhcp`

Install and configure the DHCP Server role on the DC. Authorizes the server
in AD, creates the corp lab scope with exclusions + MAC-tied reservations
for the known clients.

## Production-MSP pattern

ONE scope per subnet covers the full DHCP-managed range; exclusions carve
out a reservation block; reservations override exclusions on a per-MAC
basis. Layout for the corp `/24`:

```
.1                Gateway (static, on virbr-corp host — not DHCP)
.10               ADDC01 (static at OS install)
.50 .. .99        Reservation block (in scope, EXCLUDED from dynamic,
                  reservations punch through)
.100 .. .199      Dynamic pool
.200 .. .254      Reserved (outside scope)

Scope:       10.10.0.50 — 10.10.0.199
Exclusion:   10.10.0.50 — 10.10.0.99
Lease:       8 days (Windows default)
Reservations: CLIENT01→.50, CLIENT02→.51, UBUNTU01→.60
```

The scope spans `.50-.199` because **Windows DHCP rejects reservations
whose IPs aren't inside any defined scope** (`ERROR_DHCP_SUBNET_NOT_PRESENT`).
Reservations always override exclusions — documented MSFT behavior.

## Tasks

1. Install `DHCP` feature with management tools.
2. Post-install: create `DhcpAdministrators` + `DhcpUsers` security groups
   via `Add-DhcpServerSecurityGroup`, mark ServerManager `ConfigurationState=2`
   to silence the "Complete DHCP configuration" warning.
3. Authorize DHCP server in AD via `Add-DhcpServerInDC`.
4. Create / update IPv4 scope with active state.
5. Apply exclusion ranges (idempotent diff against current).
6. Set scope options 003 (Router), 006 (DNS), 015 (DNS Domain Name), 042 (NTP).
7. Apply MAC-tied reservations from `client_dhcp_reservations` +
   `linux_client_dhcp_reservations` group_vars (composed via hostvars in
   `defaults/main.yml`).
8. Verify end state.

## Reservation data source

Reservations are sourced from the two existing group_vars dicts (close to
where the hosts are defined in inventory):

- `inventory/group_vars/clients.yml` → `client_dhcp_reservations`
- `inventory/group_vars/linux_clients.yml` → `linux_client_dhcp_reservations`

The role composes them via `hostvars[first-host-in-group]` because Ansible
loads group_vars per-host, not at the group level itself. See
`defaults/main.yml::ad_dhcp_reservations` for the expression.

MACs accept `:` or `-` separators; the role normalizes to `52-54-00-aa-bb-cc`
(lowercase, dash-separated) before calling `Add-DhcpServerv4Reservation`.

## Implementation

All inline `ansible.windows.win_powershell` against the DhcpServer PS
module (ADR-016). No native Ansible module exists for DHCP server config.

## Idempotency

Each task reads current state before writing:
- Scope: compared field-by-field; `Set-` only if any differs.
- Exclusions: keyed by `"start|end"`, add missing + remove extra.
- Options: read each option's current value, compare, set only if differs.
- Reservations: keyed by IP, add missing / update changed MAC or Name / remove extra.

## Used by playbook

`03-configure-services.yml`.

## ADRs implemented

- ADR-007 (network layout)
- ADR-013 (DHCP on the DC at lab scale; production would tier off)
- ADR-016 (Ansible-native + inline PS where no module exists)
