# Role: `kvm_network`

Defines and starts the `corp-lab` libvirt network used by every VM in the lab.

**Scope.**
- Subnet: `10.10.0.0/24`, gateway `.1`, NAT to host.
- **libvirt DHCP is disabled** — the DC owns DHCP (ADR-007). libvirt only assigns the bridge IP.
- Bridge name: `virbr-corp`.
- MAC OUI for VMs: `52:54:00` (libvirt-allocated range).

**Idempotency.** Re-runs are no-ops once the network exists and is active.

**ADRs implemented.** ADR-005, ADR-007.

**Used by playbook.** `00-libvirt-network.yml`.
