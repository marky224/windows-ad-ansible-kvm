# Role: `ad_ntp`

Configure the PDC emulator (ADDC01) as the authoritative NTP source for the
forest. Domain-joined clients sync from the PDC automatically via NT5DS;
this role gives the PDC itself a stable upstream peer set.

## Tasks

1. Verify W32Time service is installed and Running.
2. Apply `w32tm /config /manualpeerlist:... /syncfromflags:manual /reliable:yes /update`.
3. Write `AnnounceFlags=5` (always announce + announce as reliable source) and
   `SpecialPollInterval=1024s` to registry.
4. Restart W32Time service + trigger a resync.
5. Ensure UDP/123 is open inbound (enable built-in "Windows Time" rule if
   present + create explicit fallback rule).
6. Verify `w32tm /query /source` reports one of the configured peers.

## Defaults

- Peers: `0.pool.ntp.org` through `3.pool.ntp.org` (NTP Pool Project).
  Cost: free (volunteer-operated). For production with strict SLAs, pair
  with a paid stratum-1 (NIST commercial, Meinberg appliance, etc.).
- Per-peer flag: `0x9` (`SpecialInterval | Client`).
- `AnnounceFlags`: `5` (always announce + announce as reliable).
- `SpecialPollInterval`: 1024 seconds (17m4s).

Override at playbook level via `ad_ntp_peers`, `ad_ntp_peer_flag`,
`ad_ntp_announce_flags`, `ad_ntp_special_poll_interval`.

## Why `pool.ntp.org` by default

DNS-rotating pool of community stratum-2 servers. 4 entries gives w32time
enough redundancy + outlier-rejection material. Standard MSP choice for
domains without a dedicated time appliance.

## Implementation

All inline `ansible.windows.win_powershell` wrapping `w32tm.exe` + registry
writes (ADR-016). No native Ansible module covers w32time.

## Idempotency

Reads current `w32tm /query /configuration` + registry state and only
re-applies if any value differs.

## Used by playbook

`03-configure-services.yml`.
