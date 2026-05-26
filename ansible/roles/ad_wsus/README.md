# Role: `ad_wsus`

Install WSUS on the DC for in-lab patching practice. **Fire-and-forget** sync — the initial sync takes hours, so the role kicks it off async and returns immediately. Subsequent runs check status, don't re-trigger.

**Tasks.**
1. Install `UpdateServices` Windows feature.
2. Run `PostInstall CONTENT_DIR=D:\WSUS` (or per `wsus_content_dir` var) via `wsusutil.exe`.
3. Set initial sync source: Microsoft Update.
4. Trigger first sync **async** and return.

**Implementation.** All inline `win_powershell` against the UpdateServices module (ADR-016).

**ADRs implemented.** ADR-013 (WSUS in initial build).

**Used by playbook.** `02-configure-dc.yml`.
