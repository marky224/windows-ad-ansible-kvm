# Role: `ad_admins`

Creates the named domain admin account that serves as the steady-state Ansible identity for `corp.markandrewmarquez.com`. Per ADR-032, this is the daily-use admin — the built-in `Administrator` (RID 500) is hardened to break-glass only by the `ad_harden_builtin_admin` role immediately after.

**Tasks.**
1. Create `OU=Admins,DC=corp,DC=markandrewmarquez,DC=com` (idempotent via `microsoft.ad.ou`).
2. Create `madmin-da` user in `OU=Admins` with vaulted password, set as member of `Domain Admins` + `Enterprise Admins` (idempotent via `microsoft.ad.user`).
3. Verify group memberships via `Get-ADUser -Properties MemberOf`.

**ADRs implemented.** ADR-032 (named-admin tier).

**Required input variables.**
- `named_admin` (dict from `group_vars/all/main.yml`): `sam`, `upn`, `ou_dn`, `display_name`, `description`.
- `vault_named_admin_password` (vault).

**Connection assumption.** The calling play connects as the built-in domain Administrator with `vault_dc_admin_password` — specifically `Administrator@{{ forest.fqdn }}` (UPN form, post-promotion). This role MUST complete successfully before `ad_harden_builtin_admin` locks out that identity.

**Idempotency.** `microsoft.ad.ou` + `microsoft.ad.user` are both declarative. `update_password: on_create` ensures vault rotations don't fight manual changes; to force-rotate, override at run time: `-e update_password=always`.

**Password rotation procedure.**
1. `ansible-vault edit ansible/inventory/group_vars/all/vault.yml` → set new `vault_named_admin_password`.
2. `ansible-playbook 02-configure-dc.yml --tags rotate-named-admin -e update_password=always` (the rotate-named-admin tag is set on the create-user task).
3. Note: this only works while the previous password still authenticates Ansible to the DC. If you've forgotten the old password, recover via DSRM-mode + `Set-ADAccountPassword` on the DC console.

**Future scope.** When the lab adds tier-0/tier-1/tier-2 OU separation (Phase 6+), this role splits into `ad_admins_t0`, `ad_admins_t1`, `ad_admins_t2` — each creating a tier-suffixed account (`madmin-da-t0`, etc.) in the appropriate sub-OU.

**Used by playbook.** `02-configure-dc.yml` (second play).
