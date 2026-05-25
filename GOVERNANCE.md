# Project Governance

This document describes how decisions are made in the Crypto and
Gotham projects.

**Status:** v1 — single-founder governance, with explicit plan to
evolve as the project grows.

**Last reviewed:** 2026-05-25

---

## Project structure

Crypto and Gotham are open-source projects published under AGPLv3
with a commercial dual-licensing option. They are operated by:

- **The Project Lead** — currently Angel, sole founder.
- **The Commercial Entity** — to be created (SASU, FR) before the
  first commercial contract.
- **Contributors** — anyone who has had a pull request merged.

These three roles are distinct on purpose:

- Open-source code lives under AGPLv3 and is community-owned in the
  sense that AGPLv3 cannot be revoked.
- The commercial entity holds the trademarks, the right to grant
  commercial licences, and operates any paid services.
- Contributors retain copyright on their contributions but grant a
  dual licence (see [CONTRIBUTING.md](CONTRIBUTING.md)).

---

## Decision-making

### Today (single-founder phase)

While the project has a single founder:

- **Technical decisions** (architecture, dependencies, release
  cadence) are made by the Project Lead.
- **Security-critical decisions** (cryptographic primitives, threat
  model scope, audit firm selection) require external review before
  shipping — see "External review obligations" below.
- **Commercial decisions** (pricing, customer contracts, fundraising)
  are made by the Project Lead acting as director of the Commercial
  Entity once created.

This is **not a permanent arrangement**. It reflects the current
maturity of the project (single contributor, pre-seed, pre-audit).

### Future (post-seed, post-first-hire)

As the team grows beyond one person:

- A **Technical Steering Committee (TSC)** will be formed, composed
  of senior engineers from the project plus 1-2 external advisors.
- The TSC will own technical-roadmap decisions, dependency choices,
  and release approvals.
- The Project Lead retains the casting vote in deadlocks.
- The TSC publishes minutes after each decision (transparency
  obligation).

### Future (post-GA, post-multiple-commercial-customers)

If the project sustains a commercial business and multiple paid
maintainers:

- A **Foundation or non-profit governance body** may be created to
  hold the protocol specification (Gotham) separately from the
  commercial entity.
- This separation is standard practice for projects that aim to
  become infrastructure (compare with the Sigstore / Cloud Native /
  TLS specifications model).

---

## External review obligations

The following changes MUST receive external review before merging to
`main`:

1. **Any change to a cryptographic primitive** (X25519, ML-KEM-768,
   ChaCha20-Poly1305, Ed25519, HKDF, Argon2id) — must be reviewed by
   at least one external cryptographer.
2. **Any change to the Gotham wire format** — must be reviewed by at
   least one external mixnet researcher.
3. **Any new dependency in the cryptographic hot path** — must
   pass `cargo-audit` clean AND have a documented justification in
   the PR.
4. **Any change to the threat model** — must update
   [GOTHAM-THREAT-MODEL.md](GOTHAM-THREAT-MODEL.md) and be reviewed
   by the Project Lead plus at least one external advisor.

External review can be solicited via:

- A direct request to advisors listed in `02_Team/advisors.md` of
  the internal data room.
- A public RFC opened as a GitHub Discussion (for community-visible
  changes only).
- A commissioned audit (for substantial changes).

---

## Release management

### Versioning

The project uses [Semantic Versioning](https://semver.org/) (SemVer):

- **Major** (`X.0.0`) — breaking wire-format or API change.
- **Minor** (`1.Y.0`) — new features, backwards-compatible.
- **Patch** (`1.2.Z`) — bug fixes, security patches.

For the Gotham protocol, there is an additional **protocol version**
byte (`PROTOCOL_VERSION` in `crypto-gotham/src/lib.rs`) that
increments on any wire-format change, independent of the crate's
SemVer.

### Cadence

- **Stable releases:** every 3 months (Q1, Q2, Q3, Q4 of each year).
- **Beta releases:** every month.
- **Nightly builds:** continuous (per-commit on `main`).
- **Security patches:** released on demand within the SLA in
  [SECURITY.md](SECURITY.md).

### Release approval

Each release requires:

- All CI gates green (`fmt`, `clippy`, `test`, `audit`, `doc`).
- A signed [CHANGELOG.md](CHANGELOG.md) entry.
- A signed Git tag (Ed25519 key on YubiKey).
- A signed release artefact (cosign / Sigstore).
- For minor + major releases: a release-note blog post.

---

## Conflict resolution

Disagreements about technical direction are resolved in this order:

1. **Documentation** — does the existing spec already answer the
   question? If yes, no debate needed.
2. **Discussion** — open a GitHub Discussion or Issue, gather input
   from contributors and (if relevant) external advisors.
3. **TSC decision** (once formed) — the TSC votes, simple majority,
   Project Lead has casting vote.
4. **Project Lead decision** (today) — final.

For Code of Conduct disputes, see [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).

---

## Trademark policy

The trademarks **Crypto** and **Gotham** are held by the Commercial
Entity. The names may be used freely:

- To refer to the software (e.g. "I use Crypto" or "Gotham relay
  operator").
- In academic publications, blog posts, news articles, conference
  talks — including in titles.
- To describe compatible software or services (e.g. "this client
  speaks the Gotham protocol").

The names may NOT be used:

- To name a competing or modified distribution of the software in a
  way that could cause confusion (e.g. "Crypto+ Edition" or
  "GothamPro").
- To promote services that the Commercial Entity has not authorised
  (e.g. "Crypto-as-a-Service").
- In domain names that mimic the official identity.

For commercial uses outside this policy, contact the Commercial
Entity directly.

---

## Funding and finances

The Commercial Entity may seek funding from:

- Equity investors (seed, Series A, etc.) — covered by separate
  shareholder agreements.
- Government grants (Bpifrance, Horizon Europe, etc.).
- Commercial customer revenue (subscription, on-prem licences,
  support).
- Audited bug bounty pots and security research grants.

The project will NOT accept funding from:

- State surveillance agencies, intelligence services, or military
  contractors whose primary product is offensive cyber capability.
- Companies whose business model is the sale of personal data.

This list is non-exhaustive; the Project Lead reserves the right to
decline any funding that conflicts with the project's stated values.

A summary of funding sources is published once per year in the
project's annual transparency report (planned, starting calendar
year 2027).

---

## Modification of this document

Changes to this governance document follow the same rules as any
other change in the project:

- Open an issue or RFC to discuss.
- Submit a pull request.
- Receive maintainer approval.

Substantive governance changes (TSC formation, foundation creation,
trademark transfer) additionally require:

- A public RFC with a minimum 4-week comment period.
- A signed announcement on the project blog (once it exists).

---

## Contact

- Technical: GitHub Issues / Discussions.
- Security: see [SECURITY.md](SECURITY.md).
- Conduct: see [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).
- Trademark / commercial: `legal@<production-domain>` *(placeholder)*.
- Press: `press@<production-domain>` *(placeholder)*.
- Investors: `investors@<production-domain>` *(placeholder)*.
