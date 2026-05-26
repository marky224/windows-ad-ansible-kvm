# Role: `domain_join_windows`

Join a Windows host to `corp.markandrewmarquez.com`.

**Tasks.**
1. Use `microsoft.ad.membership` to join the host to the domain with `CORP\Administrator` credentials.
2. Place the computer in the `Workstations` OU (or per `target_ou` var).
3. Reboot handled by the calling playbook.

**Idempotency.** Skips if `domain_role` is already `member_workstation` or `member_server`.

**Used by playbook.** `04-join-domain.yml`.
