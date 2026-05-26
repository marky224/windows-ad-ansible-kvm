# Role: `ad_dhcp`

DHCP server setup on the DC.

**Tasks.**
1. Install `DHCP` Windows feature.
2. Authorize the DHCP server in AD.
3. Create scope `10.10.0.100 – 10.10.0.199` with lease duration 8 days, mask `/24`, router `10.10.0.1`, DNS `10.10.0.10`, domain `corp.markandrewmarquez.com`.
4. Add MAC-tied reservations for each client in `client_dhcp_reservations` + `linux_client_dhcp_reservations` group vars (so Ansible knows client IPs before they boot).
5. Exclude `10.10.0.1 – .49` and `10.10.0.200 – .254` from dynamic pool.

**Implementation.** No Ansible-native DHCP module — all `ansible.windows.win_powershell` against the DhcpServer PS module (ADR-016).

**ADRs implemented.** ADR-007.

**Used by playbook.** `02-configure-dc.yml`.
