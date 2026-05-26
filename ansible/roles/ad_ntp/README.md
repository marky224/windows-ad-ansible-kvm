# Role: `ad_ntp`

Configure the PDC as the authoritative NTP source for the domain.

**Tasks.**
- Set `w32time` config: peer list = NIST pool / `time.windows.com`, type `NTP` (not `NT5DS`).
- Mark the PDC as a reliable time source (`announce flags = 5`).
- Open UDP 123 in Windows firewall.
- Restart `w32time` service.

**Implementation.** Inline `win_powershell` invocations of `w32tm.exe`. No native module.

**Used by playbook.** `02-configure-dc.yml`.
