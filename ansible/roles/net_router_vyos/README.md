# net_router_vyos

Builds **VYOS01**, the Phase 6 inter-site router VM, from the free VyOS **rolling**
ISO on the KVM host. Routing / DHCP-relay / netem are configured separately over
SSH by `playbooks/12-configure-vyos.yml` (the `vyos.vyos` collection).

- **Decision record:** ADR-052 (+ 2026-06-02 amendment) in `_private/docs/DESIGN-DECISIONS.md`
- **Topology / IP plan:** `_private/docs/PHASE6-MULTISITE.md` (sub-milestone **SM2**)
- **Build playbook:** `playbooks/11-provision-vyos.yml` (this role)

```
VYOS01   eth0 10.10.0.2/24  (corp-lab, NAT)        <- HQ leg,     management + egress
         eth1 10.20.0.1/24  (corp-branch, isolated) <- Branch leg, = ADDC02's default gateway
         routes 10.10 <-> 10.20 (connected) · default route via 10.10.0.1 · ~40ms netem
```

## What this role does (host-side, unprivileged)

1. Verifies the pinned VyOS ISO is staged + SHA256 matches (`iso_paths.vyos` /
   `iso_checksums.vyos`; fetch with `_private/scripts/vyos-fetch.sh`).
2. Creates a **blank** qcow2 install target.
3. `virt-install`s the domain: **SeaBIOS + i440fx** (`--machine pc`), 2 virtio NICs
   (**HQ first → eth0**, Branch second → eth1), **disk-first boot** (ADR-053:
   blank disk `boot.order=1` falls through to the ISO `boot.order=2`), no eject.

It does **not** wait for SSH — the install is interactive (below).

## Why VyOS is built differently from the other VMs (researched 2026-06-02)

| Choice | Reason |
|---|---|
| **SeaBIOS, not UEFI/OVMF** | VyOS only vendor-tests BIOS; rolling images can't Secure Boot (unsigned shim). |
| **i440fx (`pc`), not q35** | Flat PCI bus → deterministic NIC enumeration (eth0=HQ, eth1=Branch). |
| **ISO install, not `--import`** | No free VyOS qcow2 (rolling = ISO only; qcow2 is subscriber-gated). |
| **No cloud-init seed** | cloud-init can *configure* VyOS but not *install* it; for one router, a one-time console install + golden snapshot is simpler and more certain than depending on cloud-init on a rolling build. |
| **Disk-first boot (ADR-053)** | Same boot-race fix as the Windows VMs — no CD eject. |

## Interactive `install image` (one-time, then snapshot a golden base)

After `11-provision-vyos.yml` boots the VM, watch the SPICE console and drive the
installer. Capture state with `virsh screenshot VYOS01 /tmp/vyos.ppm` (readable as
an image); send input with `_private/scripts/typestr.py`.

Live login (live ISO): **`vyos` / `vyos`**, then run `install image`. The prompt
sequence on the rolling build (verify on screen — the set drifts slightly by build):

| # | Prompt (abbreviated) | Response |
|---|---|---|
| 1 | "Would you like to continue? [y/N]" | `y` |
| 2 | "name this image? (Default: …)" | Enter (accept default) |
| 3 | "password for the 'vyos' user:" | **`vault_vyos_password`** |
| 4 | "confirm password" | (same) |
| 5 | "console K(VM)/S(erial)? (Default: S)" | `K` (we drive + observe via SPICE) |
| 6 | "RAID-1 mirroring?" *(only if >1 disk)* | `n` |
| 7 | "Which drive? (Default: /dev/vda)" | Enter |
| 8 | "delete all data? [y/N]" | `y` |
| 9 | "use all free space? [Y/n]" | Enter |
| 10 | "boot config? (Default: 1)" | Enter |

Type each line, e.g.:

```bash
# login
python3 _private/scripts/typestr.py VYOS01 -- "vyos" --enter
python3 _private/scripts/typestr.py VYOS01 -- "vyos" --enter
python3 _private/scripts/typestr.py VYOS01 -- "install image" --enter
# … answer prompts, verifying each via: virsh screenshot VYOS01 /tmp/vyos.ppm
# step 3/4 password WITHOUT echoing the secret to the terminal/transcript:
VYP="$(ansible-vault view ansible/inventory/group_vars/all/vault.yml \
        | sed -n 's/^vault_vyos_password:[[:space:]]*//p' | tr -d \"'\\\"\")"
python3 _private/scripts/typestr.py VYOS01 -- "$VYP" --enter   # password
python3 _private/scripts/typestr.py VYOS01 -- "$VYP" --enter   # confirm
```

When install finishes, `reboot`. Disk-first boot means it comes back up on the
**installed** system (the ISO is never re-booted).

## Minimal console bootstrap (make it reachable for Ansible)

Log in on the console (`vyos` / `vault_vyos_password`) and apply just enough to
reach SSH — everything substantive is then done declaratively by
`12-configure-vyos.yml`:

```
configure
set interfaces ethernet eth0 address 10.10.0.2/24
set service ssh
commit
save
exit
```

Now `10.10.0.2:22` answers. Ansible connects as `vyos` (password auth, from vault)
and `12-configure-vyos.yml` installs the control-node SSH key, pins `hw-id`, sets
the Branch leg + default route + netem, and `save`s.

> Then **snapshot the golden base** per ADR-008 so rebuilds skip the install:
> `cp ~/vm-lab/disks/VYOS01.qcow2 ~/vm-lab/disks/VYOS01.qcow2.vm-built`.

## Gotchas banked from research (2026-06-02)

- **netem `delay` is a silent no-op without `bandwidth`** — set both, bind to
  egress (handled in `12-configure-vyos.yml`). Rolling uses `set qos policy
  network-emulator …`, **not** the old `set traffic-policy …`.
- **`hw-id` MAC must equal the libvirt NIC MAC** or VyOS fails config-load on
  reboot ("migrate system configure failed"). The MACs here (`vyos.hq_mac` /
  `vyos.branch_mac`) are the single source of truth for both.
- **DHCP-relay is deferred to SM7** — VyOS bug **T5679 (WONTFIX)** breaks a relay
  that uses the same interface as both listen + upstream, which reciprocal
  cross-site relay requires. SM7 leans on **Windows DHCP failover** (server-to-
  server) instead. This role/playbook intentionally configures **no** dhcp-relay.
- **`vyos_config` auto-commits but does NOT persist** — the config play ends with a
  `save: true` task or the config evaporates on reboot.
