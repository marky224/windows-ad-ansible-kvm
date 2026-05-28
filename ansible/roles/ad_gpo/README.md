# Role: `ad_gpo`

Import the MSFT Security Compliance Toolkit (SCT) **Server 2025 v2602** baseline
and apply a small lab-specific delta on top.

## Tasks

1. **Local SCT integrity check** — assert the SCT ZIP exists at
   `ansible/files/sct/<filename>` with SHA256 matching the pin in
   `defaults/main.yml`. The ZIP is gitignored (MS license + repo hygiene
   for a public portfolio repo); fetched once via `_private/scripts/sct-fetch.sh`.
2. **Stage on ADDC01** — copy ZIP to `C:\SCT-Staging\`, extract.
3. **Ensure `OU=Member Servers`** at domain root (empty in single-DC lab;
   SCT Member Server GPOs link here).
4. **Import 6 Server 2025 baseline GPOs** (Domain Controller, Domain Controller VBS,
   Domain Security, Member Server, Member Server Credential Guard, Defender Antivirus)
   via `Import-GPO -BackupId <guid> -CreateIfNeeded`. Idempotent: skips if GPO
   description matches the version-tag marker.
5. **Import 2 IE11 baselines** (Computer + User, import-only, no link). End-of-life
   product; imported for completeness so a future legacy-app deployment can link
   them quickly.
6. **Link baselines to canonical OUs:**
   - DC + DC VBS → `OU=Domain Controllers,DC=...`
   - Domain Security → domain root (account/lockout/Kerberos policies)
   - Member Server + MS Credential Guard + Defender → `OU=Member Servers,DC=...`
7. **Create "Lab Delta - Firewall Logging" GPO** with the post-prune ADR-019 delta:
   firewall log path/size/dropped-packets logging for all three profiles.
   The original 12-override list was pruned to 1 deliberate delta after the
   SCT v2602 inventory showed 10 of 12 are baseline-covered (see ADR-019
   amendment 2026-05-29).
8. **Link Lab Delta GPO at domain root.**
9. **Invoke-GPUpdate /force** to apply immediately.
10. **Verify** — every imported GPO exists in `Get-GPO -All`; every link
    appears in `Get-GPInheritance` for its target OU.

## ADRs

- **ADR-019** (amended 2026-05-29) — SCT baseline + pruned 2-override lab delta.
- **ADR-037** — inline PS (GroupPolicyDsc is an abandoned 2019 personal module).

## Prerequisites

1. **Fetch the SCT zip** once on the control host:
   ```bash
   ./_private/scripts/sct-fetch.sh
   ```
   Downloads `WindowsServer2025-SecurityBaseline-2602.zip` (~1.4 MB) + `LGPO.zip`
   (~520 KB) into `ansible/files/sct/` and verifies SHA256 against the pinned
   values in `ansible/files/sct/.version`.

2. **Domain Controller** with `ad_dc` + `ad_dns` already converged. The role
   runs as the steady-state `madmin-da` named admin, which is Domain Admins
   + Enterprise Admins per ADR-032.

## Variables

See `defaults/main.yml`. Key knobs:

| Variable | Default | Notes |
|---|---|---|
| `ad_gpo_sct_zip_filename` | `WindowsServer2025-SecurityBaseline-2602.zip` | Bump when MS releases v2606 / v2706. |
| `ad_gpo_sct_zip_sha256` | (pinned) | Must match `ansible/files/sct/.version`. |
| `ad_gpo_sct_version_tag` | `SCT-WS2025-v2602` | Description-marker stamped on each imported GPO; idempotency key. |
| `ad_gpo_baseline_imports` | 6 entries | Display name + backup GUID + link target per GPO. |
| `ad_gpo_import_ie11` | `true` | Import IE11 baselines (no link). Set `false` to skip. |
| `ad_gpo_lab_delta_name` | `Lab Delta - Firewall Logging` | The post-prune ADR-019 delta GPO. |
| `ad_gpo_firewall_log_size_kb` | `16384` | Firewall log retention (16 MB ≈ ~1 week on a low-traffic DC). |

## Server 2025 baseline gotchas to flag (from ADR-037 research)

Settings the SCT v2602 baseline enforces that commonly break legacy environments
— relax in the Lab Delta GPO only if you hit them:

1. **"Deny log on through Remote Desktop Services" Local account** — broke RDP for
   break-glass local admins in earlier baselines; v2602 narrowed scope to Guests only.
2. **NTLM Restrict: Outgoing NTLM = Deny all** — breaks legacy file shares, scanners,
   MFPs; commonly downgraded to Audit-only.
3. **LDAP signing required + LDAP channel binding required** — breaks older
   Linux/appliance LDAP binds; often relaxed to "Negotiate signing" until
   inventory is clean.
4. **SMB signing always required (both directions)** — historically breaks
   NetApp/EMC and old NAS; less of an issue in 2026.
5. **WSUS / ESU delivery hardening** — does not break upstream/downstream sync
   but blocks ESU delivery to Win10 clients (orthogonal to this lab).

## Used by playbook

`04-configure-services-advanced.yml`.
