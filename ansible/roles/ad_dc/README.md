# Role: `ad_dc`

Promotes a fresh Windows Server VM into the first DC of `corp.markandrewmarquez.com`.

**Tasks.**
1. Set static IPv4 (`10.10.0.10/24`, gw `10.10.0.1`, primary DNS `127.0.0.1`).
2. Rename host to `ADDC01-corp` (if not already).
3. Install `AD-Domain-Services` Windows feature.
4. Promote to first DC of new forest via `microsoft.ad.domain` (creates forest + domain).
5. Reboot handled by the calling playbook (so it can re-establish the WinRM connection on the new domain identity).

**ADRs implemented.** ADR-006 (forest name), ADR-012 (single-DC topology), ADR-016 (Ansible-native).

**Input variables.**
- `forest.fqdn`, `forest.netbios` (from `group_vars/all/main.yml`)
- `safe_mode_password` (DSRM, from `group_vars/dc.yml` → vault).

**Used by playbook.** `02-configure-dc.yml`.
