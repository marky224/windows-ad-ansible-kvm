# Role: `ops_backup`

Back up AD state from the DC: a Microsoft-supported **system-state** backup plus
small, human-readable **config exports**. Runs in a play targeting `hosts: dc`.

## What it does

1. **Mount sentinel** (delegated to localhost) — refuses to run unless
   `backups_dir` is a real mountpoint, so an unmounted backup drive can never
   dump backups onto the NVMe OS disk.
2. **Backup volume** (delegated to localhost, unprivileged) — `qemu-img create`
   a qcow2 on the dedicated backup drive + `virsh attach-disk` it to the DC
   (idempotent), mirroring the WSUS-disk pattern (ADR-039).
3. **Windows disk init** — rescan, then initialize/partition/format the volume
   as NTFS at `E:` (idempotent; refuses to guess if 0 or >1 candidate disks).
4. **System-state** — `wbadmin start systemstatebackup -backupTarget:E:`
   (NTDS.dit + SYSVOL + registry + AD CS), then `-keepVersions` retention.
5. **Config exports** to `C:\Backups\<UTC>` → zipped → `fetch`ed to
   `{{ backups_dir }}/<UTC>.zip` → DC staging removed → host zips pruned to
   the last N:
   - `Backup-GPO -All` (restorable GPO backups)
   - `Backup-CARoleService -DatabaseOnly` (CA DB; key is in system-state)
   - `Export-DhcpServer` (scopes/reservations/options/leases)
   - `Export-DnsServerZone` (human-readable; AD zones are also in system-state)
   - `csvde` (AD object inventory)

## Host safety

The DC is a guest; the play never targets the control host. The only host-side
work is unprivileged `qemu-img`/`virsh` + writing the backup qcow2 and fetched
zips under `backups_dir`. **The dedicated backup drive is mounted ONCE by the
operator, by hand** (see `PREREQUISITES.md` → *Dedicated backup drive*). Ansible
never edits host `/etc`, `fstab`, or `smb.conf`.

## Key variables (see `defaults/main.yml`)

| Var | Default | Purpose |
|---|---|---|
| `ops_backup_disk_size_gb` | `200` | Backup qcow2 size (holds ~7 system-state versions) |
| `ops_backup_disk_target` | `sdf` | SCSI target letter (sde = WSUS, ADR-039) |
| `ops_backup_drive_letter` | `E` | Windows drive letter for the backup volume |
| `ops_backup_keep_systemstate` | `7` | `wbadmin -keepVersions` |
| `ops_backup_keep_exports` | `7` | Timestamped export zips kept under `backups_dir` |
| `ops_backup_require_mountpoint` | `true` | Refuse to write if `backups_dir` isn't mounted |

## ADRs implemented

ADR-011 (virtiofs) → **superseded by ADR-017** (SMB) → **amended by ADR-046**:
Samba-by-Ansible rejected on host-safety grounds; system-state goes to a
dedicated **operator-mounted backup drive** via a qcow2 disk, not an
Ansible-configured host SMB share.

## Used by playbook

`backup-ad.yml`.
