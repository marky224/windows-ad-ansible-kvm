# Role: `ad_harden_builtin_admin`

Applies Microsoft Learn [Appendix D](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-d--securing-built-in-administrator-accounts-in-active-directory) account-flag hardening to the built-in domain `Administrator` (RID 500), demoting it from "daily-use admin" to "break-glass / disaster-recovery only."

**Tasks.**
1. Set `AccountNotDelegated` ("Account is sensitive and cannot be delegated").
2. Set `SmartcardLogonRequired` ("Smart card is required for interactive logon") — **side effect: AD randomizes the password to a 120-byte random value, making `vault_dc_admin_password` no longer usable for this account.**
3. Verify both flags stuck and the account is still `Enabled` (Appendix D requires the account stay enabled for forest-recovery break-glass access; it is NOT disabled).

**ADRs implemented.** ADR-033 (Appendix D account-flag hardening).

**Deferred to `ad_gpo` role:** Linking a `LAB-Appendix-D-RID500-Lockdown` GPO to the Default Domain Controllers OU + member-server/workstation OUs with Deny-logon rights for `DOMAIN\Administrator`. The account-flag hardening here is the primary control; the GPO is defense-in-depth. See `ad_gpo/README.md` once that role exists.

**Required input variables.** None (operates exclusively on the built-in Administrator account).

**Connection assumption.** The calling play connects as the named domain admin (`madmin-da@corp.markandrewmarquez.com`). The fact that the play reaches this role IS the verification that the named admin works — Ansible cannot connect as a user that doesn't exist or can't authenticate. If named-admin auth were broken, Play 2 (`ad_admins`) would have raised the alarm, OR Play 3 (this role's play) would have failed at connection setup BEFORE running any tasks here.

**Idempotency.** Reads `AccountNotDelegated` + `SmartcardLogonRequired` first; only calls `Set-ADUser` if a flag is not yet set. Reports `changed=true` only when something actually changed.

**Recovery procedure if you need RID 500 back** (e.g., disaster scenario):
1. DSRM-boot the DC: F8 at boot → Directory Services Restore Mode (or `bcdedit /set safeboot dsrepair && shutdown /r /t 0`).
2. Log in at the console with `vault_safe_mode_password` (DSRM password — see `ansible/inventory/group_vars/all/vault.yml`).
3. From an elevated PowerShell: `Set-ADUser -Identity Administrator -SmartcardLogonRequired $false -AccountNotDelegated $false`. Set a new password with `Set-ADAccountPassword -Identity Administrator -NewPassword (Read-Host -AsSecureString) -Reset`.
4. `bcdedit /deletevalue safeboot && shutdown /r /t 0` to boot back to normal mode.

This recovery path is the *only* way back to a usable RID 500 once hardened. Keep `vault_safe_mode_password` backed up to an offline / off-host store.

**Used by playbook.** `02-configure-dc.yml` (third play — must run AFTER `ad_admins`).
