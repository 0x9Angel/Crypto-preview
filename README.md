# Crypto & Gotham

An end-to-end encrypted messenger with a post-quantum-hybrid mixnet.
Built in Rust. Designed for organisations whose adversaries are paying
attention.

> **Honest preamble.**
>
> This repository is **currently private**. The code you are about to
> read is **under active development** and has **not been audited by an
> independent third party**. It has **never been deployed in
> production**. The features you see across the documentation are at
> mixed levels of completion — track-level status is documented in
> [`CHECKLIST.md`](CHECKLIST.md).
>
> The project is in a build-log phase, not a release phase. We are not
> trying to sell you anything. We are not asking for your money. We are
> writing code, testing it, auditing it, and writing down honestly what
> works and what does not.

---

## When the code goes public

The repository **will be opened to the community** when **all** of the
following are true:

1. The four work tracks documented in [`CHECKLIST.md`](CHECKLIST.md)
   are at 100%. As of 2026-05-25: A · 100%, B · 83%, C · 62%, D · 0%.
2. An independent third-party security audit (Trail of Bits / NCC /
   Quarkslab / Synacktiv-class) has been completed and the report is
   public.
3. At least one production relay has been deployed and observed for
   ≥ 90 days without security incident.
4. Test coverage clears 80% across the workspace (currently 63.56% —
   see [`docs/COVERAGE.md`](docs/COVERAGE.md) once the repo is open).

These are the same conditions described in
[`THREAT-MODEL.md`](THREAT-MODEL.md) § 10. None of them are met today.
We will not open the repository before all four are met.

There is no fixed date. Stating a date we cannot defend is exactly the
marketing pattern the project rejects. The current month-by-month
progress is visible at *(public build-log site URL — to be added once
the site is online)*.

If you want to be notified the day the repository goes public:
mail `Crypto.app.organisation@protonmail.com` with the subject
`Early-access list`.

---

## What this project actually contains

Two protocols and one application, plus the documentation and tests
needed to take them seriously.

### Crypto — the messenger

A desktop end-to-end encrypted chat application built on Tauri 2 +
React 19 + a Rust backend. Uses the Signal protocol (X3DH + Double
Ratchet) for E2E. SQLCipher AES-256 + Argon2id KDF for local storage.
Ed25519-signed audit log exports for compliance use. SSO (OIDC) and
SCIM 2.0 provisioning at the Enterprise tier.

### Gotham — the mixnet

A post-quantum-hybrid mixnet (X25519 + ML-KEM-768 per hop) with
Sphinx-style fixed-size packets, Loopix-style Poisson mixing, and a
pluggable-transport layer for hostile-network deployment (obfs4-like
+ meek-CDN HTTPS fronting). Designed for 100–300 ms median latency,
not for bulk file transfer. The Gotham specification is documented in
[`GOTHAM.md`](GOTHAM.md).

### Federation server

An optional `crypto-server` binary an organisation can deploy as a
relay point on dedicated infrastructure. **Has never been deployed
in production.** First production deployment is gated on the
third-party audit.

---

## Documentation index

Every Markdown file in the repository root is described here. If a
file is not in this index, it is either generated, internal-only, or
was just added — please open an issue and we will index it.

### Charter / governance

| File | What it is for |
|---|---|
| [`README.md`](README.md) | This file. The index. |
| [`CHANGELOG.md`](CHANGELOG.md) | Keep-a-Changelog-format history of every notable change. Tagged by scope (`crypto`, `gotham`, `relay`, `store`, etc.). |
| [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md) | Behaviour expected of contributors and maintainers. Standard Contributor Covenant-derived text. |
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | How to contribute when the repository opens — branch policy, commit-message conventions, test expectations, signing requirements. |
| [`GOVERNANCE.md`](GOVERNANCE.md) | Who decides what. Maintainer rights, escalation path, RFC process for protocol changes. |

### Legal / licensing

| File | What it is for |
|---|---|
| [`LICENSE`](LICENSE) | The project's dual license. Default track is **AGPL-3.0-or-later**. A commercial track exists for cases incompatible with AGPL — see below. |
| [`LICENSE-COMMERCIAL.md`](LICENSE-COMMERCIAL.md) | Terms of the commercial license track. **The commercial track is not yet operational** — no pricing, no signed licensees yet. The document is honest about that. |

### Security

| File | What it is for |
|---|---|
| [`SECURITY.md`](SECURITY.md) | Vulnerability-disclosure policy. Where to report, what to include, what to expect in response. Also lists active and closed advisories from `cargo audit`. |
| [`SECURITY-AUDIT.md`](SECURITY-AUDIT.md) | Full report of the 2026-05-25 internal security audit. 4 CVEs detected → 3 closed → 1 accepted-risk documented. 10 patches landed. |
| [`THREAT-MODEL.md`](THREAT-MODEL.md) | Project-wide threat model. Threat actors (T1–T4), assets, what we resist, **what we do NOT resist**, operational assumptions, known gaps, comparison with Signal and Tor. The honest one. |

### Roadmap / status

| File | What it is for |
|---|---|
| [`CHECKLIST.md`](CHECKLIST.md) | The master roadmap. Four tracks (A · Gotham protocol, B · Crypto app, C · Tests & audit, D · Deployment). Each item with its real status. The source of the percentages on the public site. |
| [`B10-MULTI-DEVICE-SPRINT.md`](B10-MULTI-DEVICE-SPRINT.md) | Detailed sprint plan for the B10 work item (multi-device sync). Five phases: design (1w) → implementation (2w) → testing (1w) → audit (3-4w) → merge (1w). |

### Gotham-specific documentation

The Gotham mixnet has its own document set because the protocol is
independently usable by any other application — not only by Crypto.

| File | What it is for |
|---|---|
| [`GOTHAM.md`](GOTHAM.md) | The Gotham protocol specification (v0.1 draft). Wire format, cryptographic operations, routing rules, relay protocol, cover-traffic policy, directory authority. The reference document a second implementation team would work from. |
| [`GOTHAM-CHECKLIST.md`](GOTHAM-CHECKLIST.md) | Gotham-specific roadmap. A1–A8 items for the protocol itself, plus hardening tracks (formal verification, fuzzing, side-channel resistance). |
| [`GOTHAM-THREAT-MODEL.md`](GOTHAM-THREAT-MODEL.md) | Mixnet-specific threat model. Complementary to the project-wide `THREAT-MODEL.md` — narrower and more technical. |
| [`GOTHAM-DEPLOYMENT.md`](GOTHAM-DEPLOYMENT.md) | How to deploy a Gotham relay in practice. Systemd unit files, firewall rules, key-rotation procedures, pluggable-transport configuration. Currently theoretical — no production deployment yet. |
| [`GOTHAM-OPSEC.md`](GOTHAM-OPSEC.md) | Operational security guidance for relay operators. What logs to keep, what logs not to keep, how to respond to subpoenas, how to detect a compromised neighbour. |
| [`GOTHAM-USERGUIDE.md`](GOTHAM-USERGUIDE.md) | End-user-facing documentation of what Gotham does and does not do, written without protocol jargon. For Crypto users curious about what the mixnet actually buys them. |
| [`GOTHAM-DOSSIER.md`](GOTHAM-DOSSIER.md) | Long-form technical dossier (~60 KB) covering the protocol design rationale, cryptographic choices, and comparison with prior mixnet research (Loopix, Mixminion, Sphinx, Katzenpost). For research-audience readers. |

### Legacy / historical

| File | What it is for |
|---|---|
| [`PITCH.md`](PITCH.md) | The original investor pitch (FR), targeting a €3M seed round against an enterprise-GA 2028 timeline. **Kept for historical reference.** The project has since pivoted to a public build-log strategy — the investor framing in this file no longer matches the current direction (see the v3 marketing site). May be removed before the repository goes public. |

---

## Per-crate documentation

Once the repository is public, every crate under the workspace will
carry its own `README.md` with a one-paragraph summary, public-API
notes, and a pointer to the relevant section of the top-level
documentation above. Until then, the per-crate `Cargo.toml` files
contain machine-readable scope notes (`description = ...`).

Workspace structure (high-level):

```
crypto-proto/         Wire-format types shared across the project
crypto-store/         SQLCipher-backed storage layer + KDF + recovery
crypto-agent/         X3DH + Double Ratchet + session manager
crypto-server/        Optional federation server (never deployed)
crypto-gotham/        Gotham mixnet protocol library
crypto-gotham-relay/  Gotham relay binary
crypto-tauri/         Desktop application (Tauri 2 + React 19)
crypto-enterprise/    SSO / SCIM / audit-log / clustering
crypto-licensor/      Offline license-signing operator tool
crypto-admin/         Admin dashboard
crypto-cli/           CLI tool for local operations
crypto-tests/         Cross-crate integration tests
```

---

## Working with this repository (when it opens)

When the repository becomes public, the contributor workflow is
documented in [`CONTRIBUTING.md`](CONTRIBUTING.md). The short version:

- Branch naming: `feature/<short-name>`, `fix/<short-name>`, `audit/<short-name>`.
- Commits: conventional-commits-adjacent — `feat:`, `fix:`,
  `security:`, `test:`, `docs:`. No Claude / Copilot co-authors in
  commit metadata.
- Tests: every behavioural change ships with a test. Cryptographic
  changes additionally ship with a property-based test or a Kani
  proof obligation.
- Reviews: two approvals from maintainers for cryptographic code,
  one approval for everything else. RFC process for protocol changes
  (see `GOVERNANCE.md`).

---

## Contact

- **General**: `Crypto.app.organisation@protonmail.com`
- **Security disclosure**: same address, subject `Security — vulnerability report` (see [`SECURITY.md`](SECURITY.md))
- **Commercial licensing inquiry**: same address, subject `Commercial licence` (see [`LICENSE-COMMERCIAL.md`](LICENSE-COMMERCIAL.md))
- **Early-access list (notified when the repository opens)**: same address, subject `Early-access list`

PGP key and `.onion` v3 mirror are pending — see `SECURITY.md` for the
current status of those channels.

---

## License

Dual-licensed: **AGPL-3.0-or-later OR LicenseRef-Crypto-Commercial**.
See [`LICENSE`](LICENSE) for which track applies to your use case,
and [`LICENSE-COMMERCIAL.md`](LICENSE-COMMERCIAL.md) for the
commercial-track terms.

By default — for individuals, OSS projects, NGOs, journalists,
research labs, and most commercial users — the AGPL track is the one
that applies.

---

Crypto and Gotham are not affiliated with any government, telecom
operator, or surveillance company.
