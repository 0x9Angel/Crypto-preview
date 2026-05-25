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
- `server` — the optional federation server
- `enterprise` — the OIDC / SCIM / audit-log enterprise tier
- `docs` — documentation
- `security` — security advisories or hardening changes

---

## [Unreleased]

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
