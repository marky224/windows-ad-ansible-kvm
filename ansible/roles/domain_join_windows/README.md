# Role: `domain_join_windows`

Join a Windows client to `corp.markandrewmarquez.com` and place its computer
object in `OU=Workstations`.

**Tasks.**
1. **Host-safety guard** — asserts the target is a declared `clients` host over
   WinRM (not the DC, not the control host) before doing anything.
2. Ensure `OU=Workstations` exists on the DC (`microsoft.ad.ou`, delegated +
   `run_once`). `CN=Computers` is a container and cannot be a GPO link target,
   so workstations get a real OU.
3. Join via `microsoft.ad.membership` using the **named domain admin**
   (`madmin-da@corp.markandrewmarquez.com`) — NOT the local admin and NOT the DC
   bootstrap admin. The module handles the post-join reboot + WinRM wait.
4. Verify `Win32_ComputerSystem.PartOfDomain` after the join.

**Idempotency.** `microsoft.ad.membership` is a no-op (and skips the reboot) when
the host is already joined to the target domain.

**Connection identity.** Pre-join the role connects as the local `Administrator`
(`group_vars/clients.yml`, `vault_client_local_admin_password`); the domain admin
credential is passed to the module to perform the join, not used for the WinRM
connection.

**Autoenrollment.** Certificate autoenrollment is delivered by the dedicated
`Lab - Autoenrollment` GPO (created by `ad_gpo`, linked at the domain root), not
by this role.

**Used by playbook.** `06-join-domain.yml`.
