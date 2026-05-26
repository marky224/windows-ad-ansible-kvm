# Role: `ad_gpo`

Establish the Group Policy baseline.

**Tasks.**
1. Download MSFT Security Compliance Toolkit (SCT) Server 2025 baseline (cached under `ansible/files/sct/`).
2. Import GPOs via `Import-GPO` (inline `win_powershell` — no Ansible-native module).
3. Link the imported GPOs at the domain level + Domain Controllers OU as appropriate.
4. Apply the 12 lab-specific overrides (relax password complexity, allow PowerShell remoting, etc.) — these are deltas captured in `files/gpo-overrides/`.

**ADRs implemented.** ADR-019.

**Used by playbook.** `02-configure-dc.yml`.
