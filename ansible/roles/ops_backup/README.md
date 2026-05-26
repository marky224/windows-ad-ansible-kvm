# Role: `ops_backup`

Run AD state backup against the DC.

**Tasks.**
1. **System state backup** via `wbadmin start systemstatebackup -backupTarget:\\<host>\corp-backup` (SMB share on the Linux host, exposed via Samba).
2. **Small targeted exports** via WinRM fetch back to the control node:
   - `csvde` AD object export
   - GPO XML reports (`Get-GPOReport -All -ReportType Xml`)
   - DHCP scope export (`Export-DhcpServer`)
   - DNS zone exports (`Export-DnsServerZone`)
3. Stamp the backup directory with a UTC timestamp.

**ADRs implemented.** ADR-011 (SUPERSEDED) → **ADR-017** (SMB host share for backups; `wbadmin` doesn't accept virtiofs targets).

**Used by playbook.** `backup-ad.yml`.
