# Role: `ops_firedrill`

Disaster-recovery **fire-drill** — restore a DC's backup/snapshot into a throwaway
sandbox on an **isolated** libvirt network, run the DC health checks against it,
then tear everything down. Driven by `fire-drill.yml`. Implements **ADR-047**.

> *An untested backup is not a backup.* This is the capstone that proves the
> `ops_backup` ([ADR-046](../../../_private/docs/DESIGN-DECISIONS.md)) and
> `ops_snapshot` ([ADR-008](../../../_private/docs/DESIGN-DECISIONS.md)) artifacts
> actually recover — without disrupting prod and **without touching the host**.

## What it does

1. **Isolated network** `corp-firedrill` (`10.10.1.0/24`, bridge `virbr-fd`) — no
   `<forward>` → no NAT, no route to corp-lab or the internet. dnsmasq pins the
   sandbox NIC to `10.10.1.10`. A separate subnet so it never collides with the
   live DC's `10.10.0.10` or virbr-corp's `10.10.0.1` on the host.
2. **Restore** the latest (or named) artifact into `{{ vm_lab_root }}/firedrill`:
   a full copy of the OS qcow2 + matching NVRAM (snapshot), and for the backup
   source a copy of the backup qcow2 too. The prod snapshots / backup are only ever
   **read**.
3. **Define + boot** a sandbox VM (`<source>-firedrill`) from a template that
   mirrors the prod DC's boot-critical shape (q35 + OVMF secure-boot pflash +
   restored NVRAM + hyperv-custom + virtio-scsi). It gets a **fresh NIC MAC**, so
   Windows DHCPs onto the isolated subnet; the `WinRM HTTPS-In` rule is
   `Profile=Any`, so the control host reaches it regardless of NLA.
4. **Verify** with the shared DC checks (`playbooks/tasks/smoke-dc.yml` — the same
   ones `99-smoke-test.yml` runs). The backup source additionally enumerates
   `wbadmin get versions` + integrity-tests the newest config-export zip.
5. **Always teardown** (pass *or* fail): destroy the sandbox VM, delete the workdir
   clones, undefine/destroy `corp-firedrill`. Host + `/mnt/dc-backups` untouched.

## Scenarios (`fire_drill_scenario`)

The sandbox lifecycle (isolated net → restore → 2-NIC boot → always-teardown) is shared;
`fire_drill_scenario` selects what the verify play does with the booted clone:

| Scenario | What it does | Gate |
|---|---|---|
| `smoke` (default) | Runs the shared DC health checks (`tasks/smoke-dc.yml`) — proves the snapshot/backup restores to a healthy DC. | DC passes smoke checks |
| `seize` (SM9, [ADR-058](../../../_private/docs/DESIGN-DECISIONS.md)) | Clones the **replica** and **actually seizes all 5 FSMO** onto it against the absent ADDC01, then rehearses the metadata-cleanup eviction (`playbooks/tasks/firedrill-seize.yml`). The executable proof of `DR-RUNBOOK.md`. | (1) all 5 FSMO on the clone, ADDC01=0, still GC; (2) ADDC01 fully evicted |

**Why `seize` is safe:** a clone is still the source DC's identity, so the seizure must
never reach the live forest. The `corp-firedrill` **network isolation is the load-bearing
control** (the clone's real IP is off-subnet/unroutable there; real DCs aren't on the net;
teardown is unconditional). genid is defense-in-depth only and is *latent* here (the live
DCs expose no `<genid>`, so there is no NTDS baseline for the safeguard to fire against —
the rehearsal proves this empirically and gates on "no USN-rollback event 2095" instead).
The seize runs under `become: runas` so a fresh logon token carries the **Schema Admins**
membership it adds (which `madmin-da`'s DA+EA does **not** grant — the seize-the-Schema-master
landmine). See `DR-RUNBOOK.md` + ADR-058 for the full landmine list.

## Sources

| `fire_drill_source` | Status | What it proves |
|---|---|---|
| `snapshot` (default) | **Validated** | The latest `ops_snapshot` boots to a healthy, isolated DC |
| `backup` | **Scaffold** | Boots a same-version base + attaches a copy of the backup qcow2 + *reports* `wbadmin get versions` (non-gating). Validation found a live-copy of the mounted backup volume isn't reliably `wbadmin`-readable; that + a full `systemstaterecovery` via DSRM are the documented next iterations (ADR-047 Decision 5). |

## Usage

```bash
# Snapshot drill of the latest DC snapshot (default):
ansible-playbook playbooks/fire-drill.yml

# A specific snapshot label:
ansible-playbook playbooks/fire-drill.yml -e fire_drill_label=m7-complete

# Backup-media drill:
ansible-playbook playbooks/fire-drill.yml -e fire_drill_source=backup

# SM9 FSMO-seize rehearsal on an ISOLATED clone of the replica (ADR-058):
ansible-playbook playbooks/fire-drill.yml \
    -e fire_drill_scenario=seize -e fire_drill_source_vm=ADDC02-corp -e fire_drill_label=sm8-complete
```

## Safety

- **Host-safety guard** (`tasks/guard.yml`): runs on `kvm_host` (local) only; the
  sandbox VM + network must be `*-firedrill`; the **source** must be a declared
  `*-corp` host; the isolated subnet must be disjoint from corp-lab.
- **Teardown guard** (`tasks/teardown.yml`, re-asserted inline): refuses any target
  that isn't an unambiguous `*-firedrill` VM / network / workdir — it can never
  destroy a `*-corp` VM, the corp-lab network, or `/mnt/dc-backups`.
- **Guaranteed cleanup**: `fire-drill.yml` is block/rescue/always — any failure
  still tears the sandbox down.

## Key variables (see `defaults/main.yml`)

| Var | Default | Purpose |
|---|---|---|
| `fire_drill_source` | `snapshot` | `snapshot` \| `backup` |
| `fire_drill_scenario` | `smoke` | `smoke` \| `seize` (SM9 FSMO-seize rehearsal) |
| `fire_drill_source_vm` | `ADDC01-corp` | Prod VM to drill (must be `*-corp`; use `ADDC02-corp` for `seize`) |
| `fire_drill_label` | `""` (newest) | Snapshot label to restore |
| `fire_drill_vm_name` | `<src>-firedrill` | Sandbox VM name (never `*-corp`) |
| `fire_drill_memory_mb` / `_vcpus` | `8192` / `4` | Sandbox compute |
| `fire_drill_network_name` | `corp-firedrill` | Isolated network |
| `fire_drill_vm_ip` | `10.10.1.10` | DHCP-reserved sandbox IP |
| `fire_drill_winrm_timeout` | `360` | Seconds to wait for sandbox WinRM |
