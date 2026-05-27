# Role: `kvm_iso_slipstream`

Patch a Windows install ISO **at source** by applying Microsoft Update Catalog
`.msu` cumulative updates to its `install.wim` via DISM `Add-WindowsPackage`,
then rebuild a bootable ISO with `xorriso`. Produces a deterministically-named
patched ISO with a recorded SHA256.

**Why this exists.** Server 2025 24H2 RTM (build 26100.1742) ships with three
documented Netlogon regressions that break first-DC promotion, fixed by
KB5060842 (June 2025) and later cumulatives. Rather than chase post-install
workarounds inside the running OS, this role applies the LCU to the ISO
itself so every downstream VM build starts from a known-patched baseline.
See **ADR-035** for the full design rationale and **ADR-034** for the
companion belt-and-suspenders workarounds that remain in the `ad_dc` role.

## How it works

```
Linux control node (localhost)                Windows DISM worker (delegate_to)
─────────────────────────────────             ─────────────────────────────────
1. Verify SHA256 of pristine ISO + each MSU
2. Loop-mount pristine ISO + rsync to staging
3. Start temp HTTP server on corp-lab gateway
                                              4. Invoke-WebRequest install.wim
                                              5. Invoke-WebRequest each MSU
                                              6. Add-MpPreference exclusions
                                              7. Mount-WindowsImage  index N
                                              8. Add-WindowsPackage -PackagePath
                                                 (DISM auto-orders the chain:
                                                  checkpoint LCU → target LCU)
                                              9. dism /Cleanup-Image
                                                 /StartComponentCleanup
                                              10. Dismount-WindowsImage -Save
                                              11. Publish read-only SMB share
12. smbclient pulls patched install.wim back
13. xorriso rebuilds patched ISO
14. Compute SHA256 → set fact slipstream_result
                                              15. Tear down SMB share (always:)
16. Tear down HTTP server (always:)
                                              17. Remove Defender exclusions
                                              18. (optional) clean workdir
```

The HTTP-out + SMB-in transport replaces ansible.windows.win_copy / fetch
(WinRM-chunked at ~5 MB/s) with LAN-native virtio-net throughput
(~100 MB/s), saving ~30 min per pipeline run.

## Inputs (required from caller)

| Variable | Example | Notes |
|---|---|---|
| `slipstream_pristine_iso` | `{{ iso_paths.windows_server_2025_pristine }}` | RTM source |
| `slipstream_pristine_iso_sha256` | `d0ef4502…` | provenance gate (ADR-018) |
| `slipstream_target_build` | `"26100.32860"` | drives output filename |
| `slipstream_msu_files` | `[{kb, filename, sha256}, …]` | order doesn't matter (DISM sorts) but list checkpoints first for clarity |
| `slipstream_wim_image_name` | `"Windows Server 2025 SERVERSTANDARD"` | DISM applies to one image inside the WIM |
| `slipstream_wim_index` | `2` | from `wim_edition.windows_server_2025` in `group_vars/all/main.yml` |
| `slipstream_output_iso` | `{{ iso_paths.windows_server_2025 }}` | written-to; idempotency keyed here |

## Inputs (optional)

| Variable | Default | Notes |
|---|---|---|
| `slipstream_output_iso_sha256` | unset | If set + file exists with matching SHA, entire pipeline is skipped. Capture from a successful run, then populate `iso_checksums.windows_server_2025` and feed it back through this variable. |
| `slipstream_dism_host` | `ADDC01-corp` | Inventory hostname of the Windows worker. Lab default reuses ADDC01 in its pre-promotion `vm-built` state (since it gets rebuilt from the patched ISO anyway). Can be pointed at a dedicated builder VM. |
| `slipstream_updates_dir` | `{{ iso_dir }}/updates` | Operator stages MSU files here once per LCU wave. |
| `slipstream_defender_exclusion` | `true` | Workaround for documented DISM Error 1812 (MsSense.exe file lock). |
| `slipstream_cleanup_dism_host` | `true` | Remove `C:\Slipstream` after success. |
| `slipstream_cleanup_linux_staging` | `false` | Default keeps `{{ vm_lab_root }}/slipstream-stage` for fast re-runs. |

## Operator runbook for MSU acquisition

The catalog URLs are short-lived signed tokens — they can't be hard-coded
permanently. Each LCU wave (~quarterly for this lab), refresh as follows:

```bash
# Fetch direct URLs from Microsoft Update Catalog (Python stdlib only):
python3 _private/scripts/msu-catalog-fetch.py KB5043080 KB5087539

# Save the file URLs from the output, then download:
cd ~/vm-lab/iso/updates/
curl -L -o <filename>.msu '<URL>'

# Verify and capture SHA256s for group_vars:
sha256sum *.msu
```

If a newer target LCU is chosen, re-run with the new KB. KB5043080 is the
foundational checkpoint LCU and stays in every chain until Microsoft
publishes a newer checkpoint cumulative (catalog UI shows preceding
checkpoints when you click Download on a given KB).

## Idempotency

- **Output ISO SHA256 match** ⇒ skip entire pipeline (PHASE A).
- **Staging dir already populated** ⇒ rsync extract is a no-op (PHASE B,
  `creates:` guard on `setup.exe`).
- **DISM workdir reuse** ⇒ if `slipstream_cleanup_dism_host: false`, re-runs
  reuse the pushed WIM + MSUs and re-mount.

## Failure handling

The DISM phase is wrapped in `block` / `always` so that:
- The HTTP server is always killed
- Any partial WIM mount is dismounted with `-Discard` (not `-Save`)
- Defender exclusions are always removed

If the role fails mid-pipeline, re-running picks up cleanly: the output ISO
isn't written, idempotency-skip remains false, and the next attempt
re-extracts / re-pushes / re-mounts.

## ADRs implemented
- **ADR-035** — DISM-slipstream cumulative updates into Server 2025 install media
- **ADR-018** — ISO/MSU provenance discipline (SHA256 gates)
- **ADR-026** — `xorriso` -graft-points pattern with `efisys_noprompt.bin`

## Used by playbooks
- `slipstream-iso.yml` (operator-triggered, not part of the 0N-* numbered build sequence)
