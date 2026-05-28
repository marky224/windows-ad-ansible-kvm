# Role: `ad_wsus`

Install Windows Server Update Services (WSUS) on the DC with content on a
dedicated `D:\WSUS` volume backed by a second virtual disk. Configure via
inline PowerShell following the [5tuk0v/ludus_wsus](https://github.com/5tuk0v/ludus_wsus)
2026-01 pattern (per ADR-037 amendment 2026-05-29 — DSC was tried and pulled
back because `UpdateServicesDsc.UpdateServicesServer` couples a 30-90 min
synchronous full sync into `Set-TargetResource` with no win_dsc async/poll
escape). Kick off the initial sync fire-and-forget.

## Deprecation framing — read this first

**WSUS was officially deprecated by Microsoft in September 2024** ([Windows IT Pro Blog announcement](https://techcommunity.microsoft.com/blog/windows-itpro-blog/windows-server-update-services-wsus-deprecation/4250436)). It is **not removed** from Server 2025: still ships in-box, still gets security patches (KB5070893 Oct 2025 / Server 2025 26100.6905), still supported through the Server 2025 product lifecycle.

The modern Microsoft-recommended path splits the WSUS workload across:
- **Microsoft Intune + Windows Autopatch** + [Windows Update for Business reports](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-overview) for clients
- **Azure Update Manager** (Arc-enrolled for on-prem) for servers
- **Microsoft Connected Cache** + Delivery Optimization for bandwidth caching

All three require tenant/subscription footprints (Intune licenses, Azure subscription, AAD/Entra). **For a single-DC airgapped-ish lab without an Intune tenant, there is no 1:1 on-prem replacement.** This role installs WSUS because:
1. It's the only on-prem option that still works in 2026.
2. Every legacy SMB this portfolio targets still runs WSUS.
3. Demonstrating WSUS knowledge is a stronger portfolio signal than skipping it.

The role + this README acknowledge the deprecation explicitly. See ADR-037 + ADR-039 for the full design framing.

## Tasks

### Phase 1: Libvirt-side disk prep (delegated to localhost)
1. Eject residual install CD media from ADDC01 (`virtio-win-0.1.271.iso` + per-VM seed ISO) — idempotent (`virsh change-media --eject`; `detach-disk` does not support CDROM devices).
2. Create 200 GB qcow2 at `vm-lab/disks/ADDC01-corp-wsus.qcow2` if absent. (Sized for day-one full sync + Default Approval Rule auto-approval — research-stated 40-80 GB steady-state was insufficient on first sync; see ADR-039 amendment.)
3. Live-attach the qcow2 as SCSI `sde` (libvirt's `target dev` letter pool is shared across buses; the SATA CDROMs already occupy sdb/sdc/sdd).

### Phase 2: Windows-side disk init
4. `Update-HostStorageCache` to surface the new disk.
5. Release CDROM mount points at or below `D:` via `mountvol /D` so the new volume can grab D:.
6. Find the new uninitialized disk by `PartitionStyle == 'RAW'` + size match (±5% tolerance), `Initialize-Disk -PartitionStyle GPT`, `New-Partition -DriveLetter D -UseMaximumSize`, `Format-Volume -FileSystem NTFS -NewFileSystemLabel WSUS`. Idempotent: skips if `D:` already exists with the expected label.

### Phase 3: WSUS feature install + postinstall
7. `ansible.windows.win_feature` installs `UpdateServices-Services` + `UpdateServices-WidDB` with management tools.
8. `ansible.windows.win_file` ensures `D:\WSUS` exists.
9. `ansible.windows.win_command` runs `wsusutil.exe postinstall CONTENT_DIR=D:\WSUS`, gated by the `creates: D:\WSUS\WSUSContent` predicate (the WSUSContent subfolder is created atomically by wsusutil on success — the most reliable idempotency check across community Ansible roles).
10. Belt-and-suspenders registry assert — `HKLM\SOFTWARE\Microsoft\Update Services\Server\Setup\ContentDir == D:\WSUS` (ADR-037 silent-no-op defense).
11. `ansible.windows.win_service` ensures `WsusService` is `Automatic + Running`.

### Phase 4: WSUS configure (inline PS, ludus_wsus pattern)
12. **Set sync source** → Microsoft Update + `SynchronizeAutomatically` + `TimeOfDay 02:00` + 1 sync/day. Opt out of CEIP / `MURollupOptin`.
13. **Category-only metadata sync** (only on never-synced WSUS, gated by `LastSynchronizationTime.Year == 1`). Populates the products + classifications catalog in 5-15 min — fast prerequisite for the next two tasks.
14. **Wait for category sync** via Ansible-side `until/retries/delay` polling `GetSynchronizationStatus() == 'NotProcessing'`. Timeout: 30 min (configurable).
15. **Configure classifications** via `Compare-Object` diff between desired list and live `GetUpdateClassifications()` on `.Id` (stable key — titles can shift). Pipeline `Set-WsusClassification` (enable) / `Set-WsusClassification -Disable` (disable). Post-write verify; fail if desired entry missing from live state.
16. **Configure products** — same pattern as classifications.
17. **Enable Default Automatic Approval Rule** — ships built-in with all WSUS installs (disabled by default), covers Critical + Security for All Computers. Idempotent toggle. Custom approval rules deferred to a future `ops_wsus_approve` role.

### Phase 5: Fire-and-forget full sync
18. Trigger `StartSynchronization()` async. Does not block the play; first full metadata sync takes 30-90 min on Server 2025. Steady-state sync runs nightly per the schedule set in step 12.

### Phase 6: Verification
19. End-to-end check: server reachable + ContentDir registry + configured products/classifications match intent. Pre-sync products/classifications resolve to the live MU catalog only after the category-only sync completes — verification accepts this and surfaces it as informational.

## ADRs

- **ADR-013** — Include WSUS in initial build.
- **ADR-037** (amended 2026-05-29) — ad_wsus uses inline PS (pivoted from DSC after live discovery that `UpdateServicesServer.Set-TargetResource` couples a 30-90 min synchronous full sync into the call with no win_dsc async/poll escape).
- **ADR-039** — WSUS content on `D:\WSUS` via a dedicated second virtual disk.

## Variables

See `defaults/main.yml`. Key knobs:

| Variable | Default | Notes |
|---|---|---|
| `ad_wsus_content_disk_size_gb` | `200` | qcow2 sparse; grows on demand. Empirically sized — first sync + auto-approval lands at ~170 GB; 200 GB gives ~30 GB headroom before cleanup is required. See ADR-039 amendment for the sizing journey. To resize later: shutdown VM, `qemu-img resize <path> <new>G`, start VM, `Resize-Partition -DriveLetter D -Size (Get-PartitionSupportedSize -DriveLetter D).SizeMax`. |
| `ad_wsus_content_disk_path` | `{{ vm_lab_root }}/disks/ADDC01-corp-wsus.qcow2` | Outside the repo per the vm-lab convention. |
| `ad_wsus_content_drive_letter` | `D` | Phase 2 step 5 frees D: by releasing CDROM letter mounts. |
| `ad_wsus_products` | 5 entries | Server 2025 + Win 11 24H2 + M365 Apps + Defender. **Win 11 product name was renamed late 2024** to `"Windows 11, version 24H2 and later"`. If the role's product/classification configure task throws "No products matched", the title strings have shifted; check `Get-WsusProduct \| Select -Expand Product \| Select Title` for current names. |
| `ad_wsus_classifications` | 4 entries | Critical / Security / Definition / Update Rollups. |
| `ad_wsus_synchronize_time_of_day` | `"02:00:00"` | Nightly scheduled sync. |
| `ad_wsus_trigger_full_sync` | `true` | Fire-and-forget initial full sync at the end of the role. Set false for shorter CI paths. |
| `ad_wsus_category_sync_timeout_minutes` | `30` | Bound on how long Ansible waits for the category-only sync. 5-15 min typical. |

## Drive letter conflicts

The DC was provisioned with multiple CD-ROMs (install media, virtio-win, seed-ISO). After `kvm_windows_vm` ejected the install media post-install, the remaining CDROMs occupied D:, E:, F:. Phase 1 ejects the residual install CD media at the libvirt level (the CDROM device stays attached but with no source). Phase 2 step 5 releases the in-Windows letter mounts via `mountvol /D` so the new qcow2 volume can grab `D:`. The empty install-CD slot at `sdb` (SATA, distinct from the SCSI `sde` we attach) does not block letter assignment.

## Server 2025 cmdlet gotchas (per ADR-037 research, May 2026)

- **`Install-WindowsFeature UpdateServices-Services` does NOT auto-run postinstall.** The Sept 2025 behavior change I initially suspected does not exist in any public source. If you see "WSUS already postinstalled" on a fresh server, the actual cause is a stale `HKLM\SOFTWARE\Microsoft\Update Services\Server\Setup\PostInstall` registry key from a previous failed run, not auto-postinstall.
- **`Set-WsusProduct` / `Set-WsusClassification` are pipeline-only.** Calling with `-Title 'Security Updates'` silently no-ops — no `-Title` parameter exists. This role pipes from `Get-WsusProduct | Where-Object { $_.Product.Title -in $desired }` to defend against this.
- **`Get-WsusProduct` / `Get-WsusClassification` return EMPTY on a never-synced WSUS** — the catalog is sync-populated. The role does a `StartSynchronizationForCategoryOnly()` first to populate the catalog before attempting product config.
- **`Invoke-WsusServerCleanup` hangs for hours-to-days on large catalogs.** Run one switch at a time, async, with explicit timeout. Not exercised by this role (cleanup is a future iteration).
- **Server 2025 WSUS blocks ESU delivery to Win10 clients** ([4sysops report](https://4sysops.com/archives/windows-server-2025-wsus-blocks-esu-updates/)). Orthogonal to a Win 11–only lab but worth flagging.

## Initial sync timing

- **Category-only metadata sync** (first run, blocking): **5-15 min** on Server 2025 against the canonical product set. Bounded to 30 min in defaults.
- **Full metadata sync** (fire-and-forget at end of role): **30-90 min** depending on bandwidth + MU response time. Runs async; Ansible play does not wait.
- **Content download** (the multi-GB part) only happens **after updates are approved**. The Default Automatic Approval Rule (enabled by the role) covers Critical + Security; those updates' content downloads start in the background after each successful metadata sync.

## Reference implementations

- **Gold-standard 2026 pattern**: [5tuk0v/ludus_wsus](https://github.com/5tuk0v/ludus_wsus) (last release 2026-01-01, explicitly supports Server 2025). This role's configure phase is modeled after `ludus_wsus/tasks/configure.yml`.
- **Classic inline-PS reference**: [jpboyce/ansible-role-wsus](https://github.com/jpboyce/ansible-role-wsus) (2019, simpler, less idempotent — uses disable-all-then-enable-listed instead of Compare-Object diff).
- **Cautionary DSC tale**: [oatakan/ansible-windows-wsus-example](https://github.com/oatakan/ansible-windows-wsus-example) (2024-04, runtime-patches DSC with an unmerged PR fork — concrete evidence that the DSC story is rotting).
- **Approval rule reference**: [Boe Prox's WSUS Automatic Approvals demo](https://github.com/proxb/Presentations/blob/master/WSUS_And_PowerShell/WSUS_And_PowerShell_Demo/5_AutomaticApprovalsDemo.ps1).

## Used by playbook

`04-configure-services-advanced.yml`.
