# Role: `ops_snapshot`

Disk-level snapshot management for lab VMs — `snapshot.yml`, `list-snapshots.yml`,
`rollback.yml` all dispatch through this one role (`ops_snapshot_action`).

Implements **ADR-008 (amended M7)**: a snapshot is `cp <disk>.qcow2 <disk>.qcow2.<label>`
(plus NVRAM), **not** a libvirt internal snapshot — those fail on these UEFI/pflash
VMs. `cp` needs no `sudo` (dynamic_ownership=1 relabels disks to `madmin` on stop).

## Actions

| Action | Playbook | What it does |
|---|---|---|
| `snapshot` | `snapshot.yml` | Quiesce → `cp` selected disks + NVRAM to `.<label>` → `qemu-img check` → restart |
| `list` | `list-snapshots.yml` | Read-only enumerate `*.qcow2.<label>` + `*_VARS.fd.<label>` |
| `rollback` | `rollback.yml` | Confirm → auto-save current to `.<label>-pre-rollback` → quiesce → restore → check → restart |

## Disk selection (ADR-008 amendment)

- **Default = OS disk only** (`{{ vm_lab_root }}/disks/<vm>.qcow2`) **+ NVRAM**.
- **Data disks opt-in** via `-e include_data_disks=true` (e.g. the 172 GB WSUS disk).
- The **backup disk is never eligible** — it lives at `/mnt/dc-backups` (ADR-046),
  outside the disks pool, and only pool disks are considered. CD-ROMs are excluded.
- NVRAM (`<vm>_VARS.fd`) is always captured (UEFI Secure Boot correctness).

## Quiesce + consistency

Graceful ACPI shutdown, polled up to `snapshot_graceful_timeout` (default 180 s),
then a `virsh destroy` fallback (crash-consistent — fine for AD's NTDS.dit). The
summary reports `clean` / `crash-consistent` / `offline`. The VM is restarted only
if it was running.

## Safety

- **Host-safety guard:** runs on `kvm_host` (local) only, against a declared
  `*-corp` VM, paths under `{{ vm_lab_root }}`; never wildcards.
- **Rollback** requires `-e confirm_rollback=yes` and auto-saves the current disks
  to `.<label>-pre-rollback` first (reversible). It restores only what was captured
  under the label — never the WSUS/backup disks unless they were snapshotted + opted in.

## Key variables (see `defaults/main.yml`)

| Var | Default | Purpose |
|---|---|---|
| `vm_name` | — | Target lab VM (required for snapshot/rollback) |
| `snapshot_label` | — | Snapshot label (`[A-Za-z0-9._-]`) |
| `include_data_disks` | `false` | Also capture/restore data disks (e.g. WSUS) |
| `snapshot_include_nvram` | `true` | Capture `<vm>_VARS.fd` |
| `snapshot_graceful_timeout` | `180` | Seconds to wait for clean shutdown before `destroy` |
| `snapshot_overwrite` | `false` | Allow replacing an existing label |
| `confirm_rollback` | `no` | Must be `yes` for rollback |
