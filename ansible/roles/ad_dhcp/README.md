# Role: `ad_dhcp`

Install and configure the DHCP Server role on the DC. Authorizes the server
in AD, creates the corp lab scope with exclusions + MAC-tied reservations
for the known clients. **Phase 6** extends the same role to the Branch site
(per-group scope on ADDC02) and adds reciprocal hot-standby **DHCP failover**
between the two DCs (ADR-056) — see the Phase 6 section below.

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
6. Set scope options 003 (Router), 006 (DNS — **both DCs, local-first**; ADR-056), 015 (DNS Domain Name), 042 (NTP).
7. **Phase 6 (SM9):** classless static routes — DHCP **option 121** (HQ scope only) —
   push the cross-site branch route (`10.20.0.0/24 → VyOS 10.10.0.2`) to HQ clients, whose default
   gateway (the host) can't reach the isolated branch. Includes the default route per RFC 3442. See the Phase 6 section.
8. Apply MAC-tied reservations from `client_dhcp_reservations` +
   `linux_client_dhcp_reservations` group_vars (composed via hostvars in
   `defaults/main.yml`).
9. Verify end state.
10. **Phase 6:** if this DC is the **Active** owner of the scope's failover relationship,
    re-replicate the scope to the hot-standby partner (`tasks/replicate.yml`; idempotent —
    see the Phase 6 section).

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
- Classless static routes (option 121, Phase 6): desired bytes computed per RFC 3442; the read-back
  (`0xHH` hex strings) is normalized to integers and compared, set only if differs; post-write verified.
- Reservations: keyed by IP, add missing / update on changed **ClientId (MAC)** only / remove extra. Name is set once on `Add`, then left to DHCP — comparing it would flap `changed` forever because DHCP overwrites the reservation Name with the client's registered FQDN once the client is online.
- Failover replication (Phase 6): pushes to the standby only when this DC is the scope's
  active owner AND its option **006/121** drift from the partner's copy; post-write verified.

## Phase 6 — multi-site (Branch scope + reciprocal DHCP failover)

The same role serves the Branch site and provides cross-site redundancy (ADR-056):

- **Branch scope (ADDC02).** `16-branch-dhcp.yml` runs this role against `dc_replica`
  with per-group overrides in `group_vars/dc_replica.yml` (scope `10.20.0.0/24`, options,
  `ad_dhcp_authorize_ip`, empty reservations). ADDC01's HQ path is unchanged — the only
  task change is var-ising the AD-authorize IP (`ad_dhcp_authorize_ip`, default
  `corp_subnet.dc_ip`).
- **Reciprocal hot-standby failover** (`tasks/failover.yml`, run by `17-dhcp-failover.yml`).
  Microsoft's "Symmetric Model": each DC is **Active** for its own scope and **Standby** for
  the partner's (`HQ-ADDC01-ADDC02` + `Branch-ADDC02-ADDC01`). Hot-standby, MCLT 1h, reserve
  5%, manual partner-down. Per-relationship params in `group_vars/{dc,dc_replica}.yml`
  (`dhcp_failover`); shared knobs `ad_dhcp_failover_{reserve_percent,mclt}` in `defaults`;
  shared secret `vault_dhcp_failover_secret` (inline-vault).
  - Runs under **`become: runas`** — `Add-DhcpServerv4Failover` contacts the partner over
    TCP 647 (a WinRM second hop).
  - Split for debuggability + secret hygiene: a visible pre-check → a `no_log`-narrowed
    `Add` → a read-back + assert (`Mode==HotStandby`, `ServerRole==Active` here).
  - `17-dhcp-failover.yml` fronts the creates with a **preflight gate** that guards DHCP
    failover's ±1-minute time-sync requirement (`Add-` hard-fails at create beyond it).

- **DNS-resilience + self-healing replication** (ADR-056 amendment). Each scope's **option 006**
  lists **both DCs, local-first** (HQ `[ADDC01, ADDC02]` in `defaults`; Branch `[ADDC02, ADDC01]`
  in `group_vars/dc_replica.yml`) so a client leased by the standby during a failover still gets a
  *live* resolver. Failover replicates **scope-level** options/reservations/leases (NOT server-level
  options) only at CREATE time + on an explicit push — so `tasks/replicate.yml` runs at the end of
  the role: when this DC is the scope's **Active** owner and its option 006/121/249 differ from the
  partner's copy, it runs `Invoke-DhcpServerv4FailoverReplication -ScopeId <id> -Force` (a
  destructive partner-overwrite, **from the active side only**) under `become: runas`, then re-reads
  the partner to verify. Idempotent — no-op when in sync / no relationship / on the standby side.

- **Cross-site reachability — classless static routes (option 121)** (SM9, ADR-058). The HQ scope
  hands clients a route to the **isolated** branch subnet (`10.20.0.0/24 → VyOS HQ leg 10.10.0.2`)
  because their default gateway is the host, which does not bridge into the isolated branch net — so
  without this, an HQ client cannot reach ADDC02, the option-006 cross-site DNS/auth failover target
  (the gap the SM9 DR drill surfaced: `tracert` died at the host with "Destination protocol
  unreachable"). Delivered via DHCP **option 121** at **scope** level so it rides failover replication.
  - **Option 121 only — not the legacy Microsoft option 249.** Win11 honours 121, and Win11 **24H2
    turned strict on DHCP option types** (a wrongly-typed option makes the client fail DHCP and land
    on APIPA). Option 121 is *predefined* on Windows DHCP as the correct `BinaryData` type, so it is
    24H2-safe; adding a custom 249 would be a needless second option (no down-level clients) and an
    avoidable type-mismatch hazard. (KB5065789's separate option-121 breakage is RRAS/VPN-specific,
    not LAN DHCP.)
  - Per RFC 3442 a client receiving option 121 **ignores option 003**, so the **default route is
    encoded into 121 too** (`0.0.0.0/0 → host`, listed first) — the documented #1 pitfall is clients
    losing their gateway/internet when it's omitted. Routes are declared as data
    (`ad_dhcp_classless_static_routes` in `defaults`); the role encodes the RFC-3442 bytes (decimal
    byte-string input; `0xHH` read-back). The Branch scope needs none (branch clients' gateway already
    *is* VyOS) → overridden to `[]` in `group_vars/dc_replica.yml`.

## Used by playbook

`03-configure-services.yml` (HQ scope on ADDC01). **Phase 6:** `16-branch-dhcp.yml`
(Branch scope on ADDC02) + `17-dhcp-failover.yml` (reciprocal failover, via `tasks/failover.yml`).

## ADRs implemented

- ADR-007 (network layout)
- ADR-013 (DHCP on the DC at lab scale; production would tier off)
- ADR-016 (Ansible-native + inline PS where no module exists)
- ADR-056 + amendment (Phase 6: reciprocal hot-standby DHCP failover + Branch scope + DHCP DNS-resilience)
- ADR-058 (Phase 6 SM9: cross-site classless static routes — option 121/249 — so HQ clients can reach the branch DC for DNS/auth failover)
