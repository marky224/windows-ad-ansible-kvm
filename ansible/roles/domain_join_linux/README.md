# Role: `domain_join_linux`

Join an Ubuntu host to `corp.markandrewmarquez.com` via `realmd`/`sssd`.

**Tasks.**
1. `apt install realmd sssd sssd-tools adcli samba-common-bin oddjob oddjob-mkhomedir packagekit krb5-user`.
2. Configure `/etc/krb5.conf` for the realm.
3. `realm join` against `corp.markandrewmarquez.com` with the domain admin credential.
4. Enable `oddjob-mkhomedir` PAM and create `/etc/sudoers.d/admins` granting `%domain admins@corp.markandrewmarquez.com` sudo.

**Idempotency.** `realm discover` + `realm list` to skip if already joined.

**Used by playbook.** `06-join-linux.yml`.
