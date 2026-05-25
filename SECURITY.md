# Security policy

Last reviewed: 2026-05-25.

## Reporting a vulnerability

> **TL;DR.** Email `Crypto.app.organisation@protonmail.com` with the
> subject `Security — vulnerability report`. Expect an
> acknowledgment within 72 hours. We will tell you honestly whether
> the issue is in scope and on what timeline a fix can land.

We take security seriously, but we are a small team building an
unfinished product. Be patient with response timelines and clear about
severity in your initial report.

### What to include

A useful disclosure email includes:

- **Affected component** — crypto-store, crypto-agent, crypto-gotham,
  crypto-server, crypto-tauri, or "frontend / webview".
- **Reproduction steps** — minimal, with a code example or test case
  when applicable.
- **Impact assessment** — what an attacker can do with the
  vulnerability, and the threat-actor tier (T1–T4 per
  `THREAT-MODEL.md` § 2) required to exploit it.
- **Suggested fix** if you have one.
- **CVE assignment** request if applicable.

### What not to include 

- **Do not** test the vulnerability on a system you don't own.
- **Do not** include personal data of third parties (other users,
  identifiers, message content).
- **Do not** post the vulnerability publicly before we have had a
  chance to respond. We apply standard 90-day responsible-disclosure
  practice — sooner if the fix is trivial, longer if the fix
  requires a protocol change.

### What you get

- **Acknowledgment** of receipt within 72 hours, including a
  preliminary severity assessment.
- **Status updates** at least every 14 days while the report is open.
- **Public credit** in the resolution commit message and the next
  release's `CHANGELOG.md`, unless you ask to remain anonymous.
- **A clear answer** if the report is out of scope or duplicates an
  existing tracked issue — we will say so directly rather than ghost
  you.

We do **not** currently operate a bug-bounty program. The commercial
side of the project is not yet operational (see
`LICENSE-COMMERCIAL.md`), and a bounty program without funding
behind it would be theatre. When a bounty program lands, it will be
announced in `CHANGELOG.md` and on the project site.

### Contact channels

- **Email**: `Crypto.app.organisation@protonmail.com`
- **PGP fingerprint**: *(production PGP key pending — track via
  `CHANGELOG.md` for the announcement)*
- **.onion v3 mirror**: *(pending — for high-risk sources who cannot
  safely route email through their network)*

### Anti-spam note

The email channel is monitored by a human, but volume is variable.
Use a clear subject line. Reports with subjects like "URGENT" or "0day
for sale" go to the bottom of the queue. Reports with subjects like
"crypto-store: integer overflow in chunk_index validation" go to the
top.

---

## Active advisories (from `cargo audit`)

Last `cargo audit` run: 2026-05-25.

### Accepted risk: `rsa` 0.9.10 — RUSTSEC-2023-0071 (Marvin Attack)

**Severity**: medium (5.9, CVSS v3.1).
**Status**: persistent — no upstream fix available.

The `rsa` crate is pulled in as a direct dependency of
`openidconnect` 4.0.1 (the SSO library used in the Enterprise tier).
The advisory describes a timing side-channel that could in theory
recover RSA private keys over a sustained MitM observation.

Mitigations in place:
- The vulnerable code path is **only reachable through the
  Enterprise-tier SSO flow**. Users without SSO configured are not
  exposed.
- The major IdPs (Google Workspace, Microsoft Entra, Okta) default to
  RSA-PSS or ECDSA for JWT signing — RSA-PKCS1 v1.5 (the vulnerable
  primitive) is opt-in or legacy in these providers.
- Exploitation requires a sustained MitM position on the SSO TLS
  channel for the order of hours of observation per recovered key bit.

This risk is documented in `SECURITY-AUDIT.md` finding H-4 and will
be removed the day **either** RustCrypto's constant-time RSA
implementation ships (tracked on RustCrypto/RSA PR #394), **or** the
SSO dependency tree is rewired to avoid the `rsa` crate entirely.

### Closed: 3 rustls-webpki CVEs

Closed on 2026-05-25 by upgrading `openidconnect` 3.5 → 4.0.1, which
brings `oauth2` 5.0, `reqwest` 0.12, `rustls` 0.23, and
`rustls-webpki` 0.103. The three CVEs that were active before the
upgrade:

- RUSTSEC-2026-0098 — Name constraints for URI names incorrectly
  accepted
- RUSTSEC-2026-0099 — Name constraints accepted for wildcard cert
- RUSTSEC-2026-0104 — Reachable panic in CRL parsing

Resolution commit: `75c0d57` on `main`. See `SECURITY-AUDIT.md`
finding H-3.

---

## Unmaintained dependencies (tracked, not exploitable)

`cargo audit` flags 22 transitive dependencies as unmaintained:

- **GTK-3 bindings** (`atk`, `gdk`, `gtk`, `gdkwayland-sys`, etc.) —
  pulled in by `tauri-plugin-dialog`. The Tauri team is migrating to
  GTK-4 in a future major release; the project will inherit the fix
  through the upstream upgrade.
- **`fxhash` 0.2.1** — pulled in by `selectors` via the HTML parser
  used in the Tauri webview. Same upstream-track resolution.
- **`proc-macro-error`**, **`rustls-pemfile`**, and the
  **`unic-char-*`** family — all transitive build-time dependencies
  with no runtime exposure.

None of these are exploitable. They are tracked here so that future
audits can verify they remain unexploitable and so that an upstream
fix is adopted promptly when it lands.

---

## Audit history

| Date | Auditor | Type | Outcome | Report |
|---|---|---|---|---|
| 2026-05-25 | Internal (Angel) | Full workspace review + cargo audit | 10 patches landed, 3 CVE closed, 1 risk accepted | `SECURITY-AUDIT.md` |

A third-party audit is **the gating event** for the project's
public production-readiness claim. See `THREAT-MODEL.md` § 10 for
the full set of conditions. No third-party audit is currently
scheduled, because pricing and timing for a Trail-of-Bits-class
engagement depend on funding milestones not yet hit.

---

## Process

- `cargo audit` is run manually before every release tag and after
  every dependency bump. CI integration is on the roadmap (Track
  4.2.6 in `CHECKLIST.md`).
- New advisories affecting the deployed dependency tree are triaged
  within 48 hours of disclosure.
- This file is updated as advisories close, as mitigations change,
  or as the dependency tree shifts.
- The full audit history (every internal or external audit's report)
  is kept under `docs/audits/` once the repo is public.

---

## Out-of-scope vulnerabilities

The following classes of issue are out of scope for security
disclosure:

- **Denial-of-service against the user's own machine** (e.g.
  "I can crash my own app by sending myself a malformed packet").
  These are bugs; report through normal channels.
- **Side-channel attacks on hardware** (Spectre / Meltdown class).
  Out of scope for any application-layer security policy.
- **Social engineering of users.** Reports of "I can phish the user
  into giving me their password" are operational concerns, not
  vulnerabilities.
- **Vulnerabilities in unsupported configurations.** The `direct`
  transport (without Tor / Gotham) is documented as testing-only and
  is not a supported deployment mode.
- **Issues in third-party WebView implementations.** WebKitGTK / WKWebView
  / WebView2 are OS-supplied components. Their vulnerabilities are
  the OS vendor's responsibility; we inherit them and patch when the
  OS patches.

If you are unsure whether your finding is in scope, send it anyway.
We will tell you honestly which side of the line it falls on.
