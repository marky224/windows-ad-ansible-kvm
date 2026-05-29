# Role: `ad_cs`

Deploy a single-tier Enterprise Root CA on the promoted DC for LDAPS, WinRM HTTPS,
client/computer authentication, and future cert needs.

**Production framing.** This role is a **deliberate lab compression** of the
production-canonical two-tier PKI (offline standalone Root + online Enterprise
Issuing CA). A real 50-200 seat MSP build runs the offline Root on a powered-off
VM that boots twice a year to re-sign the issuing CA's CRL. The lab collapses
the two roles onto one online Enterprise Root to keep the build deployable on
a single DC. See ADR-038 for the full rationale.

## Tasks

1. **Feature install** — `AD-Certificate` + `ADCS-Cert-Authority` via `ansible.windows.win_feature`.
2. **CA install** — Enterprise Root CA via inline PS calling `Install-AdcsCertificationAuthority` (RSA 4096 / SHA-256 / KSP-backed key / 10-year validity). Idempotent: skips if the registry's `CertSvc\Configuration\Active` already matches the configured CA common name.
3. **Web Enrollment install** — `ADCS-Web-Enrollment` + `Web-Server` (IIS), then `Install-AdcsWebEnrollment` to bind the `/CertSrv` + `/CertEnroll` virtual directories. Provides HTTP-reachable CDP/AIA endpoints for non-domain-joined clients.
4. **CRL period registry** — `certutil -setreg` for `CRLPeriod` (1 Week), `CRLDeltaPeriod` (1 Day), `CRLOverlap` (12 Hours). Reads canonical state from registry (per ADR-037 silent-no-op defense), restarts `CertSvc` only when values changed.
5. **CDP/AIA configuration** — `microsoft.ad.cs_authority` (native, v1.11.0+) enforces three CDP entries (local file + LDAP + HTTP) and three AIA entries (local file + LDAP + HTTP). The default LDAP entries are preserved for domain-joined client speed; HTTP entries make CRL/CA-cert validation work from Linux/macOS/external apps.
6. **First CRL issue** — `certutil -CRL` (skipped if a `.crl` file already exists in `C:\Windows\System32\CertSrv\CertEnroll`). Without this, newly-issued certs reference a CRL that does not yet exist on disk → HTTP 404 on validation.
7. **Template publishing** — `microsoft.ad.cs_template` (native, v1.11.0+) publishes the production-canonical 5-template set: `Machine` (V1 Computer), `WebServer` (V1), `User` (V1), `Workstation` (V2 Workstation Authentication), `KerberosAuthentication` (V2, modern DC cert per RFC 4556).
8. **End-to-end verification** — `certutil -ping` + `certutil -CAInfo` (asserts CA name + Enterprise Root) + `certutil -CATemplates` (asserts all 5 expected templates are published).
9. **Machine-autoenrollment template** (ADR-044) — clones the built-in `Workstation Authentication` into a lab-owned `Corp Workstation Authentication` (CN `CorpWorkstationAuthentication`) via `microsoft.ad.cs_template`'s `source_template` (native clone + OID mint + publish), then grants `Domain Computers` **Read + Enroll + Autoenroll** on the copy via inline PS (the one thing `cs_template` doesn't manage). Built-ins stay pristine. This is the piece that makes the domain-root `Lab - Autoenrollment` GPO actually issue machine certs — the built-ins grant Domain Computers only Read+Enroll, never Autoenroll.

## ADRs

- **ADR-016** — Ansible-native architecture (use `microsoft.ad.cs_authority` + `cs_template` where they fit; inline PS elsewhere).
- **ADR-037** — DSC vs inline PS evaluation: inline PS + native modules for `ad_cs` (DSC `ActiveDirectoryCSDsc` is on 2020 stable with no Server 2025 validation; `CertificateDsc` is healthy but doesn't beat the already-native `microsoft.ad.cs_template`).
- **ADR-038** — AD CS deployment shape: single-tier Enterprise Root, Web Enrollment installed, HTTP CDP/AIA published, production-canonical 5-template baseline.
- **ADR-044** — machine-cert autoenrollment via a cloned template: clone `Workstation Authentication` → `Corp Workstation Authentication` (native `cs_template`, chosen over hand-rolled raw-LDAP cloning), grant Domain Computers Autoenroll on the copy. Kept V2; V4 + TPM key attestation tracked as a future showcase.

## Variables

See `defaults/main.yml`. Key knobs:

| Variable | Default | Notes |
|---|---|---|
| `ad_cs_ca_common_name` | `{{ forest.netbios }}-ADDC01-CA` | Embedded in Issuer DN of every issued cert. Cannot be changed without rebuilding the CA. |
| `ad_cs_key_length` | `4096` | RSA. ECC requires algorithm-consistency changes throughout the chain. |
| `ad_cs_hash_algorithm` | `SHA256` | Current 2026 best-practice; SHA-384 only with P-384 ECC. |
| `ad_cs_validity_years` | `10` | CA cert lifetime. |
| `ad_cs_install_web_enrollment` | `true` | Set `false` to skip IIS + Web Enrollment install (drops HTTP CDP/AIA). |
| `ad_cs_http_pki_host` | `addc01.{{ forest.fqdn }}` | DNS name embedded in HTTP CDP/AIA URLs. Must resolve from anywhere a cert is validated. |
| `ad_cs_templates_to_publish` | 5 production-canonical CNs | See defaults file for the list + display-name mapping. |
| `ad_cs_autoenroll_templates` | `Corp Workstation Authentication` ← clone of `Workstation Authentication`, Domain Computers Autoenroll | Machine-cert autoenrollment templates (ADR-044). Each entry: `name` (new CN), `display_name`, `source_template` (built-in display name to clone), `autoenroll_principals` (groups granted Read+Enroll+Autoenroll). |

## Server 2025 gotchas (from ADR-038 research)

- **Don't use "Build from Active Directory information" for the User template.** Server 2025 24H2 has an open regression where the resulting Subject DN is empty (MS Q&A, no fix as of May 2026). Use "Supply in request" or duplicate to V2 with explicit Subject sources.
- **NTAuth chain.** Enterprise CA install auto-publishes the CA cert to NTAuth (`certutil -dspublish -f <ca.crt> NTAuthCA`). If you ever rebuild, verify this — the April 2025 KB5055523 break around NTAuth chaining was fixed in June 2025 KB5060842 but the dependency is real.
- **32-bit smart-card / CSP apps** broke in October 2025 due to forced KSP. Not a CA-side issue; flag if a downstream app needs legacy CSP.

## Used by playbook

`02-configure-dc.yml` (eventual long-term home), `03-configure-services.yml` or `04-configure-services-advanced.yml` (M5 transitional).
