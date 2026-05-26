# Role: `ad_dns`

DNS server configuration on the DC.

**Tasks.**
- Configure DNS forwarders (defaults: Cloudflare `1.1.1.1` + Google `8.8.8.8`, override via vars).
- Create reverse-lookup zone for `10.10.0.0/24` (`0.10.10.in-addr.arpa`).
- Verify primary zone for `corp.markandrewmarquez.com` exists and is AD-integrated.

**Implementation.** `community.windows` provides partial DNS modules (`win_dns_zone`, etc.); fall back to `ansible.windows.win_powershell` for forwarder + zone configuration.

**Idempotency.** Re-runs check existence before creating.

**Used by playbook.** `02-configure-dc.yml`.
