# Role: `ad_dns`

DNS server configuration on the freshly-promoted DC. Runs after `ad_dc` (which
installed the DNS Server feature and created the AD-integrated forward zone).

## Tasks

1. Sanity-check that DNS Server service is Running.
2. Configure upstream forwarders (default: Quad9 `9.9.9.9` + `149.112.112.112`).
3. Create AD-integrated reverse-lookup zone for the corp subnet
   (`0.10.10.in-addr.arpa`, forest-wide replication, secure dynamic updates only).
4. Enable aging/scavenging on AD-integrated forward + reverse zones (7-day
   refresh intervals, 168-hour scavenging cycle — MSFT-recommended defaults).
5. Verify end state (forwarders match expected, both zones AD-integrated).

## Why Quad9 by default

Quad9 (`quad9.net`) is a free recursive resolver operated by the non-profit
Quad9 Foundation. It refuses to resolve known-malicious domains using
threat-intel feeds from 18+ industry sources, providing resolver-level
malware blocking with no client agent required. GDPR-compliant, no
query-data sale, no logging of PII. Reasonable production-MSP choice when
client networks need a baseline defensive posture.

Override at playbook level via `ad_dns_forwarders`. Alternates:
- Cloudflare pair: `[1.1.1.1, 1.0.0.1]` — fastest, no filtering.
- Cloudflare + Google: `[1.1.1.1, 8.8.8.8]` — diversity insurance.
- NIST/internal stratum: depends on environment.

## Implementation

All inline `ansible.windows.win_powershell` against the DnsServer PS module
(ADR-016). The `community.windows.win_dns_*` modules predate the
AD-integrated reverse-zone story and are not reliable on Server 2025
24H2 (build `26100.32860`).

## Idempotency

Each task reads current state before writing. Re-runs are no-ops once the
target state is reached.

## Used by playbook

`03-configure-services.yml`.

## ADRs implemented

- ADR-007 (network layout / reverse zone scope)
- ADR-016 (Ansible-native with inline PS where no native module exists)
