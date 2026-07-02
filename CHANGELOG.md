# Changelog

All notable changes to Crypto and Gotham are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/).

Entries are tagged with the scope they affect:

- `crypto` — the Crypto messenger application and shared crates
- `gotham` — the Gotham mixnet protocol
- `relay` — the Gotham relay binary
- `tauri` — the desktop application layer
- `store` — the SQLCipher-backed storage layer
- `enterprise` — the OIDC / SCIM / audit-log enterprise tier
- `docs` — documentation
- `security` — security advisories or hardening changes

---

## [0.7.0] — 2026-05-30 — Gotham-only transport switch

This is the v0.7 protocol/product line. App builds are tagged in the
v2.1.x series (current internal HEAD ~`9b0c3da`). The workspace is
~31,919 LOC Rust with 338 automated test functions (`cargo test`
green across the workspace). One accepted risk remains:
RUSTSEC-2023-0071 (RSA Marvin) via `openidconnect`, on an unreachable
path, documented.

### Removed

- **BREAKING: gotham/crypto:** Tor v3, Lokinet, and the SMP
  federation transport are removed. Gotham is now the sole transport.
  Migration: existing contacts must be re-established over Gotham;
  there is no Tor/Lokinet/SMP fallback path anymore.
- **BREAKING: server:** the `crypto-server` (SMP federation server)
  and `crypto-cli` crates are deleted from the workspace. There is no
  longer any server to host, pin, or subpoena.

### Added — Gotham transport

- **gotham:** Sphinx v0.2 fixed-size 2048-byte packets with an
  X25519 + ML-KEM-768 hybrid (post-quantum) KEM per hop, over a
  Noise XK / QUIC link layer.
- **gotham:** Loopix-style Poisson mixing with continuous cover
  traffic; real user sends are now routed through the cover-traffic
  queue rather than dispatched directly.
- **gotham:** LIONESS wide-block non-malleable PRP (Anderson-Biham,
  4-round) protects the per-hop payload, replacing the earlier
  XOR / per-hop-AEAD branch and defeating tagging attacks.
- **gotham:** SURB (single-use reply blocks) enabling anonymous
  mailbox fetch.
- **gotham:** Gotham mailbox for store-and-forward offline delivery;
  deposit is performed over the mixnet and recipients are spread
  across mailbox hosts via HRW (rendezvous) hashing.
- **gotham:** decentralized peer-to-peer gossip transport — relays
  discover each other — anchored by k-of-n authority attestation.
- **gotham:** self-forming signed directory plus a directory
  authority (enroll / heartbeat / proof-of-possession liveness
  probe).

### Changed — Gotham path selection

- **gotham:** path selection now enforces GLOBAL diversity: a
  distinct operator and a distinct network (/16 for IPv4, /48 for
  IPv6) across the whole path, with entry not equal to exit.

### Added — Crypto app

- **agent:** Signal-style safety-number identity verification
  (60-digit), forced and persisted, with a verified flag and an
  alert on pinned-key change for MITM detection. This replaces the
  earlier TOFU server-key pinning (there is no server to pin
  anymore).
- **agent:** identity-key rotation and revocation implemented
  end-to-end.
- **store:** encrypted DB backup and restore with a pre-migration
  snapshot.
- **agent:** a multi-device primitive landed (SAS-verified device
  link + encrypted account-bundle transfer). Activation is
  deliberately gated and not shipped; end users remain one device
  per identity today.

### Fixed

- **gotham:** low-order X25519 points are now rejected
  (was `contributory`); closes the M-1 Sphinx low-order-points
  finding.

### Security

- **gotham:** message CONTENT protection (E2E) is solid and
  testable. Network-level anonymity is CURRENTLY THEORETICAL: a
  directory authority and 3 relays are online, but all 3 sit on a
  single /16, so the (correct) path-diversity guard refuses to build
  a route — no real message has yet transited the live network. The
  anonymity properties hold only once a live network spanning
  multiple /16s exists and an external audit has been completed;
  today they are not yet real-world proven. No external independent
  audit has occurred — only the internal audit (2026-05-25) plus
  several multi-agent adversarial reviews whose confirmed findings
  were all fixed. Non-goals are unchanged: no resistance to a global
  passive adversary, to >60% relay compromise, to endpoint malware,
  or to forced disclosure of non-ephemeral keys.

### Not shipped / roadmap

- **tauri:** obfs4 and meek-CDN pluggable transports are planned, not
  shipped; the real Gotham link layer is Noise XK over QUIC.
- **tauri:** audio/video calls (WebRTC) are scoped but deferred.
- **crypto:** mobile clients are not started; desktop only
  (Linux/macOS/Windows).
- **enterprise:** SCIM 2.0 and active/passive HA replication are
  enterprise-tier roadmap items, not delivered.

---

## [Unreleased]

### Added — 2026-05-25 session

- **security:** full internal security audit of the workspace (27,189
  LOC Rust + 7,103 LOC TS, 765 crate dependencies). Report published
  as `SECURITY-AUDIT.md`. 4 CVEs detected → 3 closed → 1 accepted-risk
  documented. 10 patches landed across the workspace.
- **docs:** project-wide threat model (`THREAT-MODEL.md`) covering
  threat actors T1-T4, assets in three tiers, what the project
  resists, what it does NOT resist (GPA, endpoint compromise, forced
  disclosure), known implementation gaps, and the conditions for the
  "production-ready" claim.
- **docs:** `LICENSE-COMMERCIAL.md` describing the commercial license
  track honestly — no pricing, no signed licensees, conditions under
  which a price list will be published.
- **docs:** `README.md` rewritten as a documentation index, with
  explicit conditions under which the source repository will open to
  the community (four-track 100% + third-party audit + production
  relay ≥ 90 days + coverage ≥ 80%).
- **store:** `MAX_FILE_TRANSFER_CHUNKS = 50_000` cap on
  `init_file_transfer` and `chunk_index < total_chunks` validation in
  `save_file_chunk`; closes the H-1 DoS finding where a peer could
  send `total_chunks = u32::MAX`.
- **server:** `Semaphore` cap at 10,000 concurrent connections + per-IP
  rate limit applied before the Noise XX handshake; closes the H-2
  finding where the rate limiter only triggered after the expensive
  DH operation.
- **server:** rate-limiter `DashMap` capped at 100,000 buckets; closes
  the M-7 IPv6-spray amplification.
- **server:** SCIM logs hash `user_name` via SHA-256 before writing;
  closes the M-8 PII-in-logs finding.
- **agent:** ratchet `skipped_keys` switched from `HashMap` to
  `VecDeque` with FIFO eviction (closes M-2 non-deterministic
  eviction) and manual `Drop` zeroizes message keys before the
  allocator releases them (closes M-3 swap-leak).
- **enterprise:** Ed25519 license verification switched from `verify`
  to `verify_strict` (closes M-4 malleability).
- **tauri:** CSP hardened with `object-src 'none'; frame-ancestors
  'none'; base-uri 'self'; form-action 'none'` (closes M-6).
- **tests:** C2 — six cross-crate integration tests in
  `crypto-tests/tests/server_integration.rs` driving a real `SmpServer`
  in-process on an ephemeral port (Noise handshake, queue creation,
  send/get round-trip, empty-queue None, MAX_MSG_SIZE enforcement,
  multi-sender model).
- **tests:** C7 — Vitest + @testing-library/react + jsdom configured
  for the Tauri frontend; 22 unit / snapshot tests in `App.test.tsx`
  covering `renderMarkdown`, `PresenceDot`, `EmojiPicker`,
  `FileDisplay`, and `getRtcConfig`. Tauri IPC modules mocked in
  `src/test-setup.ts`.

### Changed — 2026-05-25 session

- **enterprise:** `openidconnect` 3.5 → 4.0.1 (and transitively
  `oauth2` 5.0, `reqwest` 0.12, `rustls` 0.23, `rustls-webpki` 0.103).
  Closes **RUSTSEC-2026-0098**, **RUSTSEC-2026-0099**, and
  **RUSTSEC-2026-0104**. The SSO code path was refactored to match
  the new typestate API (`EndpointSet`/`EndpointMaybeSet`/
  `EndpointNotSet`) and the new explicit HTTP client.
- **LICENSE:** replaced the stub MIT text with the project's actual
  dual licence (`AGPL-3.0-or-later OR LicenseRef-Crypto-Commercial`).
  The MIT line was an artefact of an early scaffolding template and
  contradicted every SPDX header in the source tree.
- **SECURITY.md:** post-audit revision — three rustls-webpki CVEs
  marked closed (commit `75c0d57`), `rsa` 0.9.10 Marvin Attack
  documented as accepted-risk with mitigation chain.

### Security — 2026-05-25 session

- **agent / store / server / tauri / enterprise:** 10 hardening
  patches landed in commit `75c0d57`. See `SECURITY-AUDIT.md` for the
  full mapping of finding → file → patch.
- **enterprise:** one upstream CVE remains accepted —
  RUSTSEC-2023-0071 (rsa 0.9.10 Marvin Attack). No upstream fix
  available; mitigation documented in `SECURITY.md` and
  `THREAT-MODEL.md` § 6.8.

---

## [Unreleased — earlier]

### Added

- **docs:** project governance document (`GOVERNANCE.md`).
- **docs:** Code of Conduct based on Contributor Covenant v2.1
  (`CODE_OF_CONDUCT.md`).
- **docs:** contributing guide (`CONTRIBUTING.md`).
- **docs:** master application checklist (`CHECKLIST.md`) covering
  product, protocol, infra, tests, compliance, legal, marketing,
  distribution, and operations tracks.
- **security:** coordinated-disclosure policy formalised in
  `SECURITY.md` with explicit SLA (72-hour acknowledgment, 90-day
  embargo).
- **gotham-relay:** three cross-crate integration tests in
  `crypto-gotham-relay/tests/cross_crate_e2e.rs` covering
  sealed-sender end-to-end, directory-driven path selection, and
  replay-cache distinct-packet handling.
- **gotham-relay:** `ed25519-dalek` added as a dev-dependency for
  integration test fixtures.

### Changed

- **docs:** `GOTHAM-CHECKLIST.md` rewritten to reflect the actual
  shipped state of Track A items A1-A8 (separate columns for `main`
  and `gotham-v0.2`).
- **gotham:** intra-doc-link warnings cleaned across `header.rs`,
  `cover.rs`, and `lib.rs` (5 → 0).
- **gotham-relay:** intra-doc-link warnings cleaned across
  `client.rs` and `transport.rs`.

### Fixed

- **gotham:** doc-link `[hop_index]` in `header.rs` escaped to avoid
  rustdoc broken-link warning.

---

## [1.2.1] — 2026-05-23

### Added

- **tauri:** Gotham exposed as a first-class transport choice in the
  Settings → Profile panel, alongside Tor v3 and Lokinet.
- **tauri:** dev-mode 3-relay self-loop directory bootstrap helper
  for local testing without external infrastructure.
- **tauri:** `GothamPanel` React component (init / status / send /
  set_contact_pk / get_my_pk / set_enabled / get_enabled).

### Changed

- **store:** schema migration v15 — `gotham_pk_hex` columns added to
  `contacts` and `own_profile` tables (additive, NULL-safe).
- **tauri:** `send_message` and `set_own_profile` rewired to route
  via Gotham when enabled; SMP-over-Tor retained as fallback during
  the transition window.
- **security:** advisory document (`SECURITY.md`) created to track
  `rustls-webpki` 0.101.7 chain (RUSTSEC-2026-0098, -0099, -0104) and
  `rsa` 0.9.10 Marvin Attack with documented mitigation paths.

### Fixed

- **lint:** workspace clippy warnings reduced from 76 → 0.
- **gotham-relay:** `transport.rs` race condition where
  `Connection` was dropped before `send.finish()` was acknowledged by
  the peer, manifesting as flaky 3-hop sealed-send round-trip tests.

### Security

- **gotham-relay:** crate-level `deny(clippy::unwrap_used)` scoped to
  non-test code only (`cfg_attr(not(test), deny(...))`), removing a
  compile-break in the property-test harness while preserving the
  production-path panic-free guarantee.

---

## [1.2.0] — 2026-05 (early)

### Added — Gotham protocol Phase 4 (App integration)

- **tauri:** A4.1 — `gotham_init`, `gotham_status`, `gotham_send`
  Tauri commands wiring the embedded Gotham relay.
- **tauri:** A4.2 — `spawn_gotham_inbox_drainer` async task that
  subscribes to the entry-relay socket and dispatches decrypted
  payloads to the application.
- **tauri:** A4.3 — `send_message` fast-path through Gotham.
- **tauri:** A4.4 — `set_own_profile` broadcast over Gotham.
- **tauri:** A4.5 + A4.7 — transport selector + init progress events.

### Added — Gotham protocol Phase 3 (Directory)

- **gotham:** A3.1-A3.6 — `RelayDescriptor`, `DirectoryDoc`,
  `SignedDirectory` with Ed25519 signatures; `PathSelector` with
  operator and AS-diversity constraints; local cache with
  last-known-good fallback.

### Added — Gotham protocol Phase 2 (Relay binary)

- **gotham-relay:** A2.1-A2.7 — `gotham-relay` standalone binary;
  QUIC + Noise XK transport on UDP/443; `ReplayCache` with 5-min
  TTL; `PoissonScheduler` for per-hop delays; graceful shutdown +
  key rotation hook.

### Added — Gotham protocol Phase 1 (Sphinx packet)

- **gotham:** A1.1-A1.7, A1.9-A1.13, A1.15 — hybrid X25519 +
  ML-KEM-768 key encapsulation; constant-time HKDF sub-key
  expansion; Sphinx-style header with MAC chain and re-blinding;
  fixed-size 2048-byte packets; property tests via `proptest`;
  benchmarks under `benches/sphinx.rs`.

### Added — Other

- **agent:** profile system v14 with workspace UI v2 refactor.
- **tauri:** updater badge — passive probe with rail-button dot.
- **enterprise:** tier-4 reproducible builds with witness arbiter
  client.

---

## [1.1.0] and earlier

Detailed history is preserved in the Git log
(`git log --oneline --reverse`). Pre-1.2.0 development predates this
file.

---

## Conventions for this file

- Entries are grouped under headings: **Added**, **Changed**,
  **Deprecated**, **Removed**, **Fixed**, **Security**.
- Each entry is one bulleted line, starts with the scope tag in
  bold, then a present-tense description.
- The `[Unreleased]` section accumulates entries until a release
  cuts; at release time the section is renamed to the version and a
  fresh `[Unreleased]` is opened.
- Security advisories MUST cite the RUSTSEC / CVE identifier when
  applicable.
- Breaking changes MUST be called out explicitly with the prefix
  **BREAKING:** and a migration note.
