# Threat model — Crypto & Gotham

Last reviewed: 2026-05-25.
Version: 0.7 (pre-production).
Scope: the full project — desktop app, embedded relay, federation server,
Gotham mixnet, and the operational boundary between them. 

> **Honest preamble.** This is the threat model of a project that has
> **not yet been deployed in production**, **not yet been audited by an
> independent third party**, and whose cryptographic protocols are
> **partially original** (Gotham mixnet design). The document describes
> what the design is *intended to resist* — not what it has been *proven*
> to resist. Use this document to decide whether to evaluate the project
> further. Do not use it to decide whether to bet a life on the project
> today.
>
> The current honest answer to "is this safe enough for source
> protection / attorney-client privilege / dissident communication?" is:
> **not yet, because no audit has confirmed the implementation matches
> the design**. We will say otherwise the day an independent auditor's
> report says otherwise.

---

## Table of contents

1. [Scope and out-of-scope](#1-scope-and-out-of-scope)
2. [Threat actors](#2-threat-actors)
3. [Assets](#3-assets)
4. [Trust boundaries](#4-trust-boundaries)
5. [Threats — what we resist](#5-threats--what-we-resist)
6. [Threats — what we do NOT resist](#6-threats--what-we-do-not-resist)
7. [Operational assumptions](#7-operational-assumptions)
8. [Known gaps in the current implementation](#8-known-gaps-in-the-current-implementation)
9. [Comparison with Signal and Tor](#9-comparison-with-signal-and-tor)
10. [What changes when we say "production-ready"](#10-what-changes-when-we-say-production-ready)

---

## 1. Scope and out-of-scope

### In scope

- **Crypto** — the end-to-end encrypted desktop messenger (Tauri 2 +
  React 19 + Rust backend).
- **Embedded relay** — the per-user SMP server that ships inside the
  desktop binary and exposes a Tor Hidden Service.
- **crypto-server** — the optional federation server an organisation
  can deploy as a relay point on dedicated infrastructure.
- **Gotham** — the post-quantum-hybrid mixnet protocol (Sphinx-style
  packet format, X25519 + ML-KEM-768, Loopix-style Poisson mixing).
- **Local storage** — SQLCipher-backed SQLite database holding identity
  keys, ratchet state, message history, contact list.
- **Backup / recovery** — the master-password recovery code flow and
  the signed conversation export format.

### Out of scope

- The user's operating system, BIOS, firmware, and hardware. If your
  laptop is compromised at the kernel level, none of this software
  protects you.
- The user's physical security. If the adversary can take the unlocked
  device, decrypted message history is readable from disk.
- Other applications running on the same machine. Crypto does not
  sandbox itself from other userland processes — a malicious userland
  process running with the user's privileges can read Crypto's
  decrypted state.
- Third-party identity providers (when SSO is used). If your Okta /
  Entra / Google Workspace tenant is compromised, the SSO identity
  binding is compromised.
- The trust store of system root CAs (used by the SSO HTTPS path).
- Side-channel attacks on the hardware (Spectre / Meltdown class).

---

## 2. Threat actors

We model four tiers of adversary. Each tier inherits the capabilities of
the previous one.

### T1 — Casual snooper

- ISP, café WiFi, university network operator, neighbour with Wireshark.
- Capability: passive observation of network traffic, sometimes active
  injection (captive portals, MitM proxies).
- Goal: opportunistic data harvest. Not specifically targeting you.
- **The full project resists T1.**

### T2 — Targeted observer

- A private investigator, a small competitor, an organised crime group,
  a nation-state without GPA capability.
- Capability: targeted surveillance of a small number of endpoints,
  ability to compromise one or two relays out of dozens, ability to
  inject targeted ads / phishing.
- Goal: identify *who* talks to *whom*, when, and how often.
- **The project is designed to resist T2** under the assumption that the
  user's device itself is not compromised.

### T3 — Local-scale state adversary

- A police force or intelligence service with subpoena power over local
  infrastructure (in particular over centralised SaaS messengers),
  ability to compel a single relay operator within their jurisdiction,
  ability to perform local-scale traffic correlation (within a city,
  within a country).
- Capability: full T2 plus subpoena, lawful interception, cross-border
  cooperation within an alliance, ability to operate a moderate fraction
  of relays (say, 10–30% of the pool, but only inside their
  jurisdiction).
- Goal: targeted unmasking of a specific user; metadata graph
  reconstruction at city or country scale.
- **The project is designed to resist T3** for metadata, with caveats
  listed in §6.

### T4 — Global Passive Adversary (GPA)

- A signals-intelligence agency with global submarine-cable taps and the
  ability to observe a majority fraction of the world's IP traffic in
  near-real-time. The NSA, the FSB, GCHQ, MSS, in their published
  doctrines.
- Capability: full T3 plus global observation, plus the ability to
  passively correlate flows across continents.
- Goal: dragnet metadata collection, targeted deanonymisation of
  individuals of interest.
- **The project does NOT resist T4 in the general case.** No deployed
  mixnet does. Gotham raises the bar against T4 (cover traffic, Poisson
  mixing, fixed-size packets) but does not promise resistance.

---

## 3. Assets

### Tier-1 — Catastrophic if compromised

- **A1 · Long-term identity keys** (Ed25519 + X25519). Holds the user's
  identity across sessions. Compromise allows the adversary to
  impersonate the user for the lifetime of the key.
- **A2 · Master password** (and the operating key derived from it via
  Argon2id). Decrypts the entire local database.
- **A3 · Recovery code** (B2 feature). 32 hex characters, 128 bits of
  entropy. Compromise allows full database decryption equivalent to A2.
- **A4 · Ratchet state** for each active conversation. Compromise allows
  decryption of past and current messages with that contact.

### Tier-2 — High impact if compromised

- **B1 · Message content** (plaintext, post-decrypt, in RAM). Visible to
  any process running with the user's privileges.
- **B2 · Contact list** (names, public keys, fingerprints). Reveals the
  social graph.
- **B3 · Audit log** (Ed25519-signed, SHA-256 chained). Tampering
  invalidates the chain; export is detectable.
- **B4 · Backup envelope** (sealed under recovery wrapping key).
  Compromise without the recovery code is computationally infeasible;
  compromise with the recovery code is equivalent to A3.

### Tier-3 — Metadata, network-observable

- **C1 · Who talks to whom** (the social graph). Inferable from
  network-flow analysis if the mixnet is not used.
- **C2 · When users are online**. Inferable from connection presence.
- **C3 · Message size and frequency distribution.**
- **C4 · IP addresses connecting to known relays.**

The Gotham mixnet is specifically designed to make C1-C4 expensive for
T2 and T3 adversaries — not for T4.

---

## 4. Trust boundaries

```
┌────────────────────────────────────────────────────────────────┐
│  USER DEVICE (trusted, by necessity)                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Crypto desktop app                                      │  │
│  │  ├─ React webview ←── IPC ──→ Tauri Rust backend         │  │
│  │  ├─ SQLCipher DB on disk (encrypted at rest)             │  │
│  │  └─ Embedded SMP server (listens on Tor Hidden Service)  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                 │
│                              │ Tor SOCKS5 (or direct, or Gotham) │
│                              ▼                                 │
└──────────────────────────────┼─────────────────────────────────┘
                               │
                               │  PUBLIC NETWORK (hostile)
                               │
┌──────────────────────────────┼─────────────────────────────────┐
│  TRANSPORT LAYER             ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Tor v3 (default today)  │  Gotham (pre-alpha)            │  │
│  │  Lokinet (optional)      │  Direct (testing only)         │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────┼─────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────┐
│  RECIPIENT DEVICE (trusted by user, opaque to anyone else)     │
└────────────────────────────────────────────────────────────────┘
```

**The boundary that matters most**: between the user's device and the
public network. Everything inside the device must be considered
"trusted by necessity" — there is no software-only defense against a
compromised endpoint. The mixnet protects what crosses the boundary;
the local DB encryption protects what stays.

---

## 5. Threats — what we resist

### 5.1 Message content (E2E layer)

- **Tampered ciphertext** → caught by ChaCha20-Poly1305 / AES-256-GCM
  AEAD tags. Decryption fails closed.
- **Replayed messages** → caught by the Double Ratchet's message
  counter. Replays decrypt to a key that is then advanced; the next
  legitimate message in the same chain fails.
- **Forward-compromise of past messages** → resisted by the Double
  Ratchet's per-message key advancement. Compromise of the current
  ratchet state cannot reverse-derive past message keys.
- **Backward-compromise of future messages** → resisted by the DH
  ratchet step. After a single round-trip, the next-generation keys
  are independent of the compromised material.
- **Quantum harvesting today / decrypt tomorrow** → resisted at the
  Gotham layer (X25519 + ML-KEM-768 hybrid KEM). E2E layer is X3DH +
  Double Ratchet; not post-quantum at the application layer yet.

### 5.2 Local-storage layer

- **Disk image dump while device powered off** → resisted by SQLCipher
  AES-256-CBC at rest, keyed via Argon2id (64 MiB, t=3, p=4) over the
  master password.
- **Brute-force of the master password** → made expensive by Argon2id
  (memory-hard, GPU-resistant) plus a per-process throttle that locks
  out unlock attempts with exponential backoff (5 s → 30 s → 2 min →
  10 min).
- **Forgotten password recovery** → 128-bit recovery code, sealed
  ChaCha20-Poly1305 envelope of the operating key, stored on the same
  device as the encrypted DB. **The user must store the recovery code
  off-device** (paper, password manager not on this machine). If both
  the DB and the recovery code are on the same machine, both fall to
  the same compromise.
- **Cold-boot attacks on RAM after lock** → partial. Keys are zeroized
  on lock (B-tier hardening). Whether this is sufficient depends on
  the OS's memory-zeroization guarantees and on physical access
  considerations.

### 5.3 Transport / metadata layer (when Gotham is used)

- **Passive observation of who talks to whom** → resisted by sealed-
  sender envelopes (sender identity is inside the AEAD).
- **Passive correlation of flow timing** → resisted by Loopix-style
  Poisson mixing (per-hop random delay drawn from an exponential
  distribution).
- **Compromise of a single relay** → resisted by the multi-hop
  construction. A single relay sees only its predecessor and its
  successor; no relay sees the full path.
- **Replay attacks at the network layer** → caught by a 5-minute
  HMAC-based replay cache per relay.
- **Packet-size fingerprinting** → eliminated by fixed-size 2048-byte
  packets and cover traffic. All Gotham packets look identical on the
  wire.
- **Quantum decryption of recorded mixnet traffic** → resisted by the
  hybrid X25519 + ML-KEM-768 KEM at each hop.

### 5.4 Server / federation layer

- **Compromise of a single relay operator** → cannot decrypt traffic,
  cannot deanonymise senders (sealed-sender). Can drop traffic for the
  queues it hosts (availability, not confidentiality).
- **Subpoena of a single relay operator's logs** → reveals only the
  encrypted queue contents and the immediate neighbours' IPs. The
  re-encryption layer means even encrypted queue contents differ from
  what the sender produced.
- **Server-side tampering with stored messages** → caught by the
  AEAD authentication on the message keys (the server cannot forge a
  valid ciphertext without the message key, which it never has).

### 5.5 Identity layer

- **Forgery of a contact's identity** → resisted by Ed25519 signature
  on the signed prekey at the X3DH bundle level. The first contact
  with a peer is still subject to TOFU (trust on first use); after the
  first successful exchange, the identity is pinned and a mismatch
  raises a security alert.
- **Identity-key disclosure under coercion** → partially mitigated by
  the ephemeral nature of session keys (X3DH consumes one-time prekeys;
  ratchet keys are forward-secret). Long-term identity disclosure
  compromises *future* sessions, not retroactively past ones.

---

## 6. Threats — what we do NOT resist

### 6.1 Global Passive Adversary (T4)

A nation-state with the ability to observe a majority fraction of the
world's network traffic in near-real-time **can correlate flows**
between Gotham entry and exit relays through statistical timing
analysis, given enough samples. Mixnets raise the cost of this attack
(more samples needed, more compute needed) but do not eliminate it.

If you are an asset of interest to such an adversary, you should not
rely on this software — or on any deployed software available today.

### 6.2 Majority-relay compromise

If more than ~60% of the active Gotham relay pool is hostile (operated
by, or coerced by, the same adversary), the multi-hop anonymity
guarantee degrades. This is a known limitation of all mixnet designs.
The mitigation is **operational**: route through a diverse pool of
relay operators across jurisdictions, and reject paths that don't span
at least two trust domains. This logic is documented but not yet
implemented in the path-selection code (see `THREAT-MODEL.md` §8).

### 6.3 Endpoint compromise

Any malware running with the user's privileges on the user's device
can read decrypted messages, the contact list, and the unlocked
ratchet state. This is true of every E2E messenger ever shipped.
Defenses are operational (don't run untrusted code, use full-disk
encryption, separate work and personal devices) — not cryptographic.

### 6.4 Forced disclosure of non-ephemeral key material

If you are physically or legally compelled to surrender your master
password or your recovery code, the entire database is decryptable.
**There is no deniable-encryption layer.** A duress code that wipes or
reveals a decoy database has been considered but not implemented; the
threat-model analysis is that duress codes generally fail in practice
(the adversary asks for *the real one*, not the first one you provide).

### 6.5 Traffic analysis below state scale

A targeted observer who can position themselves on both the sender's
and the recipient's local network (e.g. by compromising both ISPs)
can correlate connection timing at the source-network and
destination-network ends, regardless of how many mixnet hops sit in
between. The Gotham cover-traffic generator makes this harder but
does not eliminate it for an attacker with full local visibility on
both ends.

### 6.6 Long-term reidentification through writing style

Even with perfect anonymous transport, message **content** can
identify a writer through stylometric analysis. This is out of scope
for any messenger and falls under operational hygiene (don't write
the same way you write your public blog when communicating
anonymously).

### 6.7 Compromise of an SSO identity provider (Enterprise tier)

If your Okta / Entra / Google Workspace tenant is compromised, the
attacker can impersonate any user that authenticates through SSO. The
local cryptographic identity (Ed25519 keypair) is a separate trust
domain and is not affected by IdP compromise — but the contact-display
name and the SSO-bound directory mapping are.

### 6.8 The 1 open upstream CVE (RSA Marvin)

`rsa` 0.9.10 is pulled in as a transitive dependency of `openidconnect`
for the SSO flow. RUSTSEC-2023-0071 (Marvin Attack) describes a timing
side-channel that could in theory recover RSA keys over a sustained
MitM observation. **No upstream fix is available** as of 2026-05-25.

Reachability is limited:
- Only enterprise-tier SSO touches this code path.
- IdPs that sign with RSA-PSS (Google, Microsoft, Okta default) are
  not in the vulnerable timing path.
- Exploitation requires a sustained MitM position on the SSO TLS
  channel for hours of network observation per recovered key bit.

This is **documented as an accepted risk** in `SECURITY-AUDIT.md`. It
will be removed the day RustCrypto/RSA-PR#394 (constant-time RSA)
ships, or the day we replace the SSO library with one not depending on
the `rsa` crate.

---

## 7. Operational assumptions

These are not cryptographic assumptions — they are things the user
must do for the threat model to hold.

1. **The recovery code is stored off-device.** Paper in a safe,
   password manager on a different machine, fireproof envelope at a
   trusted third party. NOT on the same laptop as the encrypted DB.
2. **The master password is high-entropy.** The app enforces ≥ 12
   characters across ≥ 2 character classes. This is a floor, not a
   target. Use a passphrase ≥ 20 characters if you can type it.
3. **The device's full-disk encryption is enabled.** SQLCipher is
   defense in depth; FDE is the first line.
4. **Tor or Gotham is enabled.** The "direct" transport mode is for
   testing only and exposes IP-level metadata.
5. **OS-level updates are applied.** Tauri rides on the system's
   WebView (WebKitGTK on Linux, WebView2 on Windows, WebKit on macOS).
   OS-level WebView vulnerabilities affect the app.
6. **The user verifies fingerprints out-of-band.** TOFU is the default
   trust-establishment mode. For high-stakes contacts, verify the
   contact's Ed25519 fingerprint through a separate channel (in
   person, by reading it over an authenticated voice call).
7. **The user does not run as root / administrator.** Privilege
   separation is part of the host-level threat model.

---

## 8. Known gaps in the current implementation

This list is **exhaustive** as of 2026-05-25. Anything not on this list
is *implemented* — but "implemented" still means "not third-party
audited".

### 8.1 Application

- **Multi-device sync (B10)** — single-device today. A user with two
  laptops cannot share a contact list and message history between them.
  Sprint plan in `B10-MULTI-DEVICE-SPRINT.md`.
- **Audio / video calls (B12)** — WebRTC signalling is scoped but not
  shipped. Deferred to a post-GA release.
- **Mobile clients** — iOS and Android are not started. Desktop only.
- **Duress / deniable mode** — not implemented (see §6.4).

### 8.2 Mixnet

- **Production relay deployment** — `crypto-server` has never been
  deployed in production. Local-development only. The first relay
  deployment is gated on the third-party audit.
- **Path-selection diversity logic** — the current implementation
  picks 3-5 hops uniformly from the relay pool. A jurisdiction-aware
  selection algorithm (reject paths that don't cross at least 2 trust
  domains) is documented in `GOTHAM-CHECKLIST.md` Track A.6 but not
  yet enforced.
- **v0.1 Sphinx slot leak** — the header carries a 1-byte hop_index in
  cleartext that leaks the hop's position in the path (bounded by
  MAX_HOPS = 5). The v0.2 shift-and-pad construction removes this
  leak; v0.1 is documented as a known limitation.
- **v0.1 small-order X25519 points** — `blind_alpha` does not reject
  the 8 small-order points of the Montgomery curve. A malicious
  upstream relay could substitute α to predictable shared secrets for
  downstream hops. Documented in `SECURITY-AUDIT.md` finding M-1.
  v0.2 fixes this.
- **Cover-traffic budget tuning** — the Loopix-style cover traffic is
  implemented but the rate is hard-coded. A production deployment will
  need per-user budget tuning and a feedback loop on observed mix
  fullness.

### 8.3 Tests & audit

- **Coverage at 63.56% — target 80%** before the project claims
  production readiness. The gap is concentrated in `crypto-server`
  (server never deployed, low test pressure) and the Tauri command
  layer (Playwright UI tests not yet wired — Track C.5).
- **No third-party audit yet.** Self-audit completed
  2026-05-25 (`SECURITY-AUDIT.md`) — 10 patches landed, 3 CVE closed.
  Independent audit (Trail of Bits / NCC / Quarkslab / Synacktiv-class)
  is the gating event for the production-readiness claim.
- **Fuzz harness partial.** 5 targets on the `gotham-v0.2` branch; not
  yet merged to `main` and not yet run for the standard 24 h continuous
  pass per target.
- **Formal verification partial.** 6 Kani proofs scaffolded on
  `gotham-v0.2`; the proof obligations need to be widened to cover
  `sealed_unseal` and the ratchet invariants.

### 8.4 Operations

- **No published PGP key** for security disclosures. The `SECURITY.md`
  email channel is real but not cryptographically authenticated.
- **No published .onion v3 mirror** for sensitive sources. Planned but
  not deployed.
- **No public CI run** of `cargo audit` on a pinned schedule. Done
  manually pre-release; CI integration is on the roadmap.
- **The commercial-licence track is not operational** (see
  `LICENSE-COMMERCIAL.md`).

---

## 9. Comparison with Signal and Tor

For honesty, here is where this project stands relative to the two
most-deployed comparables.

### vs Signal

| Property | Signal | Crypto |
|---|---|---|
| E2E encryption | X3DH + Double Ratchet | Same |
| Sealed-sender | Yes | Yes |
| Mixnet for metadata | No (centralised Twilio + AWS) | Yes (Gotham, when deployed) |
| Phone number required | Yes | No |
| Self-hosted possible | No | Yes |
| Audit history | Multiple third-party audits, 10+ years | None yet |
| Production traction | 100M+ users | 0 users |
| Deployed | Yes | No (pre-production) |

**Honest reading**: Signal's cryptography is the same primitive set
plus equivalent forward-secrecy guarantees. The difference is in
**metadata** (Signal's central provider sees who-talks-to-whom, Crypto
is designed not to) and in **deployment maturity** (Signal is a decade
ahead). For most users, Signal today is safer than Crypto today,
because deployment maturity matters more than design elegance until
the audit lands.

### vs Tor

| Property | Tor | Gotham |
|---|---|---|
| Topology | Onion routing | Sphinx mixnet |
| Latency target | 800-2000 ms median | 100-300 ms target |
| Mixing | None (low-latency) | Loopix Poisson |
| Post-quantum | No | Hybrid X25519 + ML-KEM-768 |
| Cover traffic | None | Configurable |
| Relay pool | ~7000 volunteer relays | 0 deployed |
| Audit history | 20 years | None |

**Honest reading**: Tor is the production-ready solution today. Gotham
is a design that targets a different optimisation point (latency +
metadata + PQ) but has not yet been operationally validated. Crypto
uses Tor as the production transport today; Gotham is enabled in dev
mode behind a feature flag.

---

## 10. What changes when we say "production-ready"

The following conditions must ALL be true before the project's
public documentation can describe it as "production-ready":

1. **Third-party cryptographic audit** of `crypto-gotham`,
   `crypto-agent`, `crypto-store`, by a recognised auditor
   (Trail of Bits, NCC Group, Quarkslab, Synacktiv, or equivalent).
   The audit report must be publicly available.
2. **Independent reproduction** of at least the critical primitives
   (Gotham hybrid KEM, X3DH key derivation, ratchet state machine) by
   a second implementation team. This may be informal (a research
   group, a sister project) but must be documented.
3. **Coverage ≥ 80%** across the workspace (currently 63.56%).
4. **All TODO items in this threat model § 8 closed**, with the
   exception of items explicitly marked as accepted risks.
5. **First production relay** deployed and running for ≥ 90 days
   without security incident.
6. **Open-source repository** publicly accessible (the GitHub repo is
   currently private during stealth-mode build-out).

None of these conditions are met today. We will not describe the
project as "production-ready" before all six are met. We will not
omit conditions from this list to bring the date forward.

---

## Document version history

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-04-12 | Initial scaffold (Gotham-only, predecessor of this file) |
| 0.5 | 2026-05-23 | Project-wide expansion, threat actor tiers |
| 0.7 | 2026-05-25 | Post-audit revision (3 CVE closed, accepted-risk documented) |

This document is reviewed every quarter and after every CVE event
that affects the deployed dependency tree. The next scheduled review
is 2026-08-25.
