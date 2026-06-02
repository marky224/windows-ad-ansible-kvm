# Role: `ad_dc_replica`

Promotes a fresh Windows Server VM into an **additional (replica) DC** of the existing `corp.markandrewmarquez.com` forest — the **Branch-Site** controller, `ADDC02` (`10.20.0.10`). Phase 6 SM5 (ADR-052 / ADR-055).

Contrast `ad_dc`, which creates the **first** DC of a **new** forest. This role joins an **existing** domain as a writable replica + Global Catalog.

**`tasks/main.yml` — promotion (run by `13-promote-addc02.yml` Play 1, as LOCAL Administrator).**
1. Sanity-check Windows `ComputerName` is `ADDC02`.
2. Install `AD-Domain-Services` with management tools (`win_feature`) — also brings RSAT-AD-PowerShell + `repadmin`/`dcdiag` for verification.
3. Pre-stage the Server 2025 NLA-Public durability registry keys (carried from `ad_dc`).
4. Ensure an inbound ICMPv4-echo firewall rule (codifies the rule applied ad-hoc in SM2; aids SM5/SM8/DR ping checks).
5. **Ensure the NIC registers its addresses in DNS** (`RegisterThisConnectionsAddress=$true`). The build's Autounattend ships NICs with registration *disabled*; a replica must re-enable it or Netlogon can't register its DC-locator records (see the correctness note below).
6. Promote via `microsoft.ad.domain_controller` (`state=domain_controller`, `site_name=Branch-Site`, `install_dns=true`, `read_only=false`, explicit `replication_source_dc=ADDC01.<fqdn>`, **`reboot=false`**).

**`tasks/verify.yml` — ADDC02-LOCAL verification (run by Play 2, as `madmin-da`, after the reboot).**
`wait_for_connection` → poll `Get-ADDomainController` → assert **writable GC in Branch-Site** → poll clean **inbound** replication (ADDC02←ADDC01) → `repadmin` evidence (local) → wait for **SYSVOL DFSR event 4604** + `SYSVOL`/`NETLOGON` shares (+ `SysvolReady` reg) → assert core services (incl. `DFSR`) Running → confirm **krbtgt AES posture** (Server 2025 secure-channel due-diligence) → **`nltest /dsregdns`** nudge + confirm the `<dsa-guid>._msdcs` CNAME resolves and `dcdiag /test:Connectivity` passes → `dcdiag /q` (local gate).

**`tasks/verify_outbound.yml` — OUTBOUND verification (run by Play 3, as `madmin-da`, LOCALLY ON ADDC01).**
Force + confirm ADDC01 cleanly inbound-replicates **from** ADDC02 (`repadmin /replicate ADDC01 ADDC02 <NC> /force` to skip the 15-min inter-site schedule; `Get-ADReplicationPartnerMetadata`) → `repadmin /showrepl` evidence → `dcdiag /q` (local ADDC01 gate).

**Why outbound is a separate play on ADDC01 — the WinRM NTLM double-hop (the SM5 finding, ADR-055).** ADDC02's `madmin-da` session is an **NTLM network logon**; it cannot make a *second* authenticated hop to ADDC01. So any `repadmin`/`Get-ADReplication*`/`dcdiag /e` that **binds to the remote DC from ADDC02** returns `Win32 110` / "not authenticated" / `DsBindWithCred… Access denied` — even though replication is healthy (the DRA uses the DC **machine account's** Kerberos, not the session). "Does ADDC01 pull from ADDC02" is ADDC01-side state, so it must be read **on ADDC01**. Each DC's state is verified on that DC; the playbook never cross-binds.

**Why both replication directions + the DNS nudge (MS KB 825036 / 275278, SM5 research).** A DC must not point its DNS client at itself until **both inbound and outbound** replication are verified, or it becomes a DNS "island" — so SM5 gates the SM6 cross-pointing on both. A fresh DC's GUID-based `_msdcs` CNAME lags registration (dcdiag `Connectivity` fails transiently); `nltest /dsregdns` is the MS-recommended fix, so the role actively nudges + confirms. The krbtgt AES check affirmatively rules out the Server 2025 RC4-krbtgt post-promo sign-in failure (here N/A: forest born on 2025 → AES-capable krbtgt, and replication + WinRM auth already work — confirmed, not assumed).

## Three correctness points that distinguish a replica from a first DC

1. **NO first-DC init-sync bypass.** `ad_dc` sets NTDS `Repl Perform Initial Synchronizations`=0 so the *first* DC (which has no replication partner) doesn't wait forever at boot. A **replica must wait to inbound-sync from ADDC01** — carrying that key would let ADDC02 advertise as a DC before it holds the directory data (lingering-object / USN-rollback risk). This omission is the single most important difference. (See `_private/docs/PHASE6-MULTISITE.md` risk note.)

2. **`reboot: false` + a manual fire-and-forget reboot, spanning an identity switch.** ADDC01's RID 500 is hardened `SMARTCARD_REQUIRED` (ADR-033); that flag replicates, so once ADDC02 is a DC the domain `Administrator` can't password-auth, and the pre-promo **local** `Administrator` no longer exists (local SAM removed at promotion). The module's `reboot:true` would try to reconnect as that now-invalid local account. So Play 1 promotes + triggers the reboot **as local Administrator** (still valid pre-reboot), and Play 2 reconnects + verifies **as `madmin-da`**. Same local→named-admin handoff as `02-configure-dc.yml`, but spanning the promotion reboot.

3. **Re-enable NIC DNS registration (the SM5 root-cause finding).** `kvm_windows_vm`'s Autounattend ships every NIC with `EnableAdapterDomainNameRegistration=false` + `DisableDynamicUpdate=true` (so a VM doesn't spam dynamic updates at an unreachable DNS during the offline/isolated install). The **first DC** gets registration re-enabled along its DNS-install path; a **replica does not**. Left as-is, post-promo Netlogon fails with event **5782 "No DNS servers configured for local system"** and registers *none* of its locator records (`_msdcs` CNAME + SRVs) — which both fails `dcdiag` Connectivity and prevents ADDC01 from resolving ADDC02's `_msdcs` CNAME to bind it for **outbound** replication. The role re-enables `RegisterThisConnectionsAddress` before promotion. (Diagnosed via `netlogon.log` + an ADDC01-vs-ADDC02 settings diff; ADR-055.)

## DNS stays pointed at ADDC01 (deliberate)

This role does **not** touch ADDC02's DNS client (still → `10.10.0.10`). A fresh replica pointed at *itself* before its AD-integrated zones replicate in is the classic "DNS island". DNS cross-pointing is **SM6**, done only after replication here verifies green.

**ADRs implemented.** ADR-052 (multi-site topology), ADR-055 (replica promotion shape).

**Required input variables.**
- `forest.fqdn` — from `group_vars/all/main.yml`.
- `named_admin.upn` + `vault_named_admin_password` — the Domain Admin credential that authorizes the promotion.
- `safe_mode_password` (→ `vault_safe_mode_password`) — DSRM password, from `group_vars/dc_replica.yml`.
- Role knobs in `defaults/main.yml` (site name, install_dns, replication source, reconnect/verification windows).

**Connection assumption.** Play 1 must override `ansible_user: Administrator` (local form) + `ansible_password: "{{ vault_dc_admin_password }}"`. Play 2 uses the steady-state `group_vars/dc_replica.yml` identity (`madmin-da` UPN).

**Idempotency.** This is a **one-shot build** path — not idempotent across the identity switch (after a successful promotion Play 1's local-Administrator identity can no longer connect). To re-verify a built replica, run only Play 2's tasks; to redo, roll back to the pre-promo snapshot. Same one-shot nature as `02-configure-dc.yml`.

**Used by playbook.** `13-promote-addc02.yml`.
