# ops_teardown

Destroy lab `*-corp` VMs and their **live** disks — the inverse of the `00→07`
provisioning chain. Host-side, **unprivileged** (`virsh` + file deletes under
`~/vm-lab/disks` only). Driven by `playbooks/teardown.yml`. See **ADR-048**.

## Safe by default

A run only **previews** unless you arm it with `-e confirm=DESTROY`:

```bash
# PREVIEW — print exactly what would die, change nothing:
ansible-playbook playbooks/teardown.yml

# ARM — tear down the whole *-corp fleet + their live disks:
ansible-playbook playbooks/teardown.yml -e confirm=DESTROY
```

A non-empty `confirm` that is **not** the token is a hard error (never a silent
no-op — see `[[feedback-powershell-silent-noop-defenses]]`).

## What it does

For each target VM: `virsh destroy` (force off) → `virsh undefine --nvram` →
delete the disk images **attached to the domain** (resolved via `virsh
domblklist`, filtered to the disks pool) → delete the NVRAM file.

## Targeting

| Invocation | Targets |
|---|---|
| _(no `vm_name`)_ | the whole declared `*-corp` fleet (`groups.all` ∩ `*-corp`) |
| `-e vm_name=CLIENT01-corp` | that single declared `*-corp` host |

## Scope flags (default = VMs + live disks only)

| Flag | Default | Effect |
|---|---|---|
| `-e teardown_include_network=true` | off | also `net-destroy`/`net-undefine` `corp-lab` (recreate with `00-libvirt-network.yml`) |
| `-e teardown_purge_snapshots=true` | off | also delete the `.<label>` snapshot copies that sit beside the live disks in the pool |

## Always preserved

- **`/mnt/dc-backups`** — the dedicated backup drive (ADR-046). It lives *outside*
  the disks pool, so `domblklist` never even surfaces it; teardown cannot touch it.
- **`.<label>` snapshots** — preserved unless `teardown_purge_snapshots=true`.
  They are siblings in `~/vm-lab/disks`; teardown deletes only the disks *attached*
  to each domain, so the labeled copies survive a default run.

## Host safety

Mirrors the `ops_snapshot` / `ops_firedrill` guard style
(`[[feedback-never-touch-control-host]]`, the #1 project invariant):

- runs on `kvm_host` with a **local** connection, never a guest;
- every target must match `^[A-Za-z0-9-]+-corp$` **and** be a declared inventory
  host — no wildcards, never `localhost`/`all`;
- the per-VM `teardown-vm.yml` re-asserts those invariants **inline** so no loop
  iteration can act on anything else;
- `assert` neither connects nor escalates, so a mis-target aborts before any
  `virsh`/file delete.

## Rebuild

After a teardown, rebuild with `playbooks/site.yml` (or the `00→99` chain). If you
kept the snapshots, you can also restore a labeled state with
`playbooks/rollback.yml` once the VMs are re-provisioned.
