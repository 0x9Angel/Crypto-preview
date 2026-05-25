# Commercial license — Crypto & Gotham

**SPDX-License-Identifier:** `LicenseRef-Crypto-Commercial` (messenger)
**SPDX-License-Identifier:** `LicenseRef-Gotham-Commercial` (mixnet)

Last reviewed: 2026-05-25. 

> **Honest status, read first.**
>
> The commercial license track described below **is not yet operational**.
> No commercial licensee has been signed. No price list has been
> published. No purchase channel exists. This document exists so that
> commercial users can plan ahead — *not* so we can pretend to be open
> for business.
>
> If you need a commercial license today, contact us. We will tell you
> honestly whether the work to issue one is realistic on your timeline.
> Pricing, terms, and signature will be set on a per-engagement basis
> until a published price list lands.
>
> Contact: `Crypto.app.organisation@protonmail.com`.

---

## Why a commercial track exists

The default open-source track is **AGPL-3.0-or-later** (see `LICENSE`).
The AGPL adds, on top of the GPL, a clause that turns **network use of
modified software** into a **distribution event** — meaning if you run a
modified fork as a hosted service, you must publish the source of that
modification under the AGPL.

That trigger is **intentional**. It exists to prevent a third party from
taking the AGPL code, hosting it as a closed SaaS, and capturing all the
value without contributing back. The project's threat model assumes the
network-effect adversary, not just the technical one.

The **commercial track** exists for one specific case: organisations
whose use is **incompatible with the AGPL source-disclosure trigger**,
typically because:

1. They ship a closed-source product that embeds or links against the
   library form of the code (e.g. integrating `crypto-agent` or
   `crypto-gotham` as a dependency of a proprietary application).
2. They operate a hosted service whose customer agreements **forbid**
   them from open-sourcing the deployment-specific code paths.
3. They are bound by procurement rules (e.g. defense, critical
   infrastructure) that prohibit AGPL software from being used in
   covered systems regardless of disclosure intent.

For everyone else — and that includes individuals, OSS projects,
research labs, journalists, NGOs, public-sector bodies, and most
commercial users — **the AGPLv3 track is sufficient and recommended.**

---

## What the commercial license is intended to grant

The eventual commercial license — when it is operational — is intended
to grant:

- **Use** of the software in commercial products and services without
  triggering the AGPLv3 source-disclosure clause.
- **Modification** of the source for purposes covered by the agreement.
- **Distribution** of the resulting software as part of a larger
  commercial offering, under the licensee's own terms with respect to
  end users.
- **Redistribution rights** clearly delimited per agreement (sub-license,
  resale, white-label).

The exact extent of each right will be set in the per-engagement
agreement. There is no "one-size-fits-all" template yet, because the
project is too early-stage to commit to one.

---

## What the commercial license does NOT grant

- **No warranty.** The same NO WARRANTY notice that applies to the AGPL
  track applies to commercial licensees. This is non-negotiable and
  reflects the current state of the codebase (pre-production, no
  third-party audit). Customers needing warranty-backed software should
  not adopt this project yet.
- **No SLA.** No service-level agreement exists. The project has no
  hosted infrastructure to deliver an SLA against.
- **No support.** Commercial licensees do not receive a separate support
  channel beyond what the public issue tracker offers. If support
  becomes part of a future commercial offering, it will be priced
  separately.
- **No exclusivity.** Commercial licensees cannot prevent the project
  from licensing the same code to others, including their competitors.
- **No trademark license.** "Crypto" and "Gotham" are project names; the
  commercial license covers the code, not the right to call your fork
  by the same name.
- **No patent license beyond Section 11 of the AGPLv3.** If the project
  authors discover a patent infringement in the code in the future,
  commercial licensees are protected under the same terms as AGPL users.

---

## Honest scope of "commercial use" under the AGPL

A common misconception is that **AGPL forbids commercial use**. It does
not. You can:

- Use the AGPL version inside your company without any disclosure
  obligation, as long as you are not redistributing the software or
  exposing it to users over a network.
- Charge money for support, training, consulting, or hosting around the
  AGPL version, with no obligation to switch to the commercial track.
- Build a closed-source application that **runs on the same machine** as
  an AGPL-licensed daemon (e.g. an internal tool that talks to a local
  Gotham relay) without triggering the AGPL clauses — the AGPL trigger
  applies to *modifications of* the AGPL code, not to *separate
  programs that use it as a network service*.

The commercial track only becomes necessary when you cross into one of
the scenarios listed in **"Why a commercial track exists"** above.

If you are unsure, **ask before you build**. We will give you a straight
answer about whether your case requires a commercial license. Many
"obviously commercial" use cases turn out to be covered by the AGPL.

---

## Process (when the commercial track becomes operational)

When the commercial track is operational, the process will be:

1. **Inquiry.** Email `Crypto.app.organisation@protonmail.com` with a
   short description of intended use, organisation, and timeline.
2. **Fit check.** We will tell you whether your use case actually
   requires a commercial license, or whether the AGPL track is
   sufficient. About half of inquiries land here.
3. **Terms negotiation.** For genuine commercial cases, a per-engagement
   licence agreement is drafted. Pricing depends on scope (seats, hosted
   vs embedded, redistribution rights, support level).
4. **Signature.** Both parties sign. The licensee receives a signed
   license certificate referenced by the licensor's Ed25519 key (see
   `crypto-enterprise/src/license.rs`).

Until the track is operational, **inquiries are still welcome** — we
will respond honestly with the current state and a realistic timeline.

---

## Trigger for the published price list

A public price list will be published when **two** conditions are met:

1. The third-party security audit (Trail of Bits / NCC / Quarkslab /
   Synacktiv-class) is complete and published.
2. At least one production deployment exists (i.e. someone other than
   the maintainers running the software seriously).

Neither condition is met today. We will not publish a price list before
both are. This is a deliberate choice — pricing a pre-audit, pre-deploy
product is dishonest because no one can rationally compare it to its
competitors.

---

## Compatibility table

| Use case | License track |
|---|---|
| Individual desktop user | AGPL |
| Self-hosted relay for personal use | AGPL |
| OSS project depending on `crypto-proto` | AGPL |
| Research / academic use | AGPL |
| NGO operating an internal deployment | AGPL |
| Newsroom operating an internal deployment | AGPL |
| Public-sector body operating an internal deployment | AGPL |
| Internal-only commercial use (no network exposure to customers) | AGPL |
| Closed-source product embedding `crypto-agent` as a library | Commercial |
| SaaS reselling the software as-a-service without source disclosure | Commercial |
| Defense / critical-infra procurement forbidding AGPL | Commercial |
| Government deployment with code-escrow requirement | Commercial |

If your scenario does not appear above, contact us — we will add it
honestly to the table.

---

## Contact

Email: `Crypto.app.organisation@protonmail.com`
PGP fingerprint: *(not yet published — a production PGP key is pending)*
Onion mirror: *(not yet published — a production .onion v3 is pending)*

These contact channels are real but rate-limited and may take several
days to respond. The project is run by a small team and answers are
typed by humans, not auto-generated.

---

*This document does not constitute a binding offer to license the
software on any specific commercial terms. Until a signed
per-engagement agreement is in place, the only license under which you
may use this software is the AGPL-3.0-or-later track described in
`LICENSE`.*
