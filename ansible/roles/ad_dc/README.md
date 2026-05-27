# Role: `ad_dc`

Promotes a fresh Windows Server VM into the first DC of a new forest (`corp.markandrewmarquez.com`).

**Tasks.**
1. Sanity-check Windows `ComputerName` is `ADDC01` (set earlier by Autounattend's specialize pass — not renamed here).
2. Install `AD-Domain-Services` Windows feature with management tools (`win_feature`).
3. Promote to first DC of the forest via `microsoft.ad.domain` (creates forest + domain in one shot, installs DNS Server, reboots inline).
4. Wait for `Get-ADDomain` + `Get-ADForest` to respond after reboot (up to `ad_dc_ready_poll_minutes`, default 5).
5. Run `dcdiag /q` and surface any failed tests.
6. Verify `NTDS`, `DNS`, `Netlogon`, `Kdc`, `W32Time` services are Running.

The role does NOT create the named domain admin — that's the `ad_admins` role's job (ADR-032). It does NOT harden the built-in Administrator — that's `ad_harden_builtin_admin` (ADR-033). Both are separate plays in `02-configure-dc.yml`.

**ADRs implemented.** ADR-006 (forest name), ADR-012 (single-DC topology), ADR-016 (Ansible-native), ADR-031 (single-password lifecycle).

**Required input variables.**
- `forest.fqdn`, `forest.netbios` — from `group_vars/all/main.yml`.
- `safe_mode_password` — DSRM password, from `group_vars/dc.yml` → `vault_safe_mode_password`.

**Connection assumption.** The calling play must override `ansible_user: Administrator` (local form, NOT UPN, NOT `CORP\Administrator`) + `ansible_password: "{{ vault_dc_admin_password }}"` at the play level. This is because at the moment of promotion, the local Administrator is the only working identity. Post-promotion, the same account becomes the domain Administrator with the same password (per ADR-031). Subsequent plays in `02-configure-dc.yml` use UPN form.

**Idempotency.** `microsoft.ad.domain` returns `changed=false` if the host is already a DC of the named domain. Validation tasks always run (they're informational, not state-changing).

**Used by playbook.** `02-configure-dc.yml` (first play).
