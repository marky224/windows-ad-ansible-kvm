# files/

Static assets used by roles. Anything dynamic (per-VM seed ISO contents, rendered templates) lives elsewhere and is gitignored.

**Expected contents** (added as roles are implemented):
- `sct/` — Microsoft Security Compliance Toolkit GPO baselines (downloaded once, cached here).
- `gpo-overrides/` — the 12 lab-specific GPO deltas (ADR-019).
- (No `ConfigureRemotingForAnsible.ps1` — removed upstream July 2023; WinRM bootstrap is inline in Autounattend per ADR-021.)

**Gitignored** (per `.gitignore`):
- `rendered/` — per-VM rendered Autounattend / cloud-init contents
- `*-seed.iso` — per-VM seed ISOs built by `xorriso`
