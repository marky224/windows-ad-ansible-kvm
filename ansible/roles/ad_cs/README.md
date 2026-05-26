# Role: `ad_cs`

Stand up an Enterprise Root CA on the DC for LDAPS + WinRM HTTPS + future cert needs.

**Tasks.**
1. Install `AD-Certificate` + `AD-Certificate-Authority` features.
2. Create the root CA via `microsoft.ad.cs_authority` (native since v1.11.0 — partly replaces pure-PS approach).
3. Publish baseline templates via `microsoft.ad.cs_template` (Computer, WebServer, User).
4. Optionally issue a cert for the Ansible control node so we can tighten `winrm_server_cert_validation` from `ignore` to `validate`.

**ADRs implemented.** ADR-016 (Ansible-native AD CS where possible).

**Used by playbook.** `02-configure-dc.yml`.
