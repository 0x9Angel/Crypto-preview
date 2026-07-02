# Gotham — Master Checklist

> All tracks in one document. Tick `[x]` as items complete. Each section
> ends with a `Definition of Done` so we don't ship half-baked.

**Last updated:** 2026-07-03
**Project lead:** Angel
**Protocol design:** Angel

---

## TRACK A · Protocol implementation (Rust)

### Phase 1 — Sphinx packet (3 weeks)

- [x] **A1.1** `hybrid::encapsulate(rng, x25519_pk, mlkem_pk) → (α, α', shared)` (X25519 + ML-KEM-768 hybrid KEM per hop)
- [x] **A1.2** `hybrid::decapsulate(x25519_sk, mlkem_sk, α, α') → shared`
- [x] **A1.3** Constant-time HKDF expansion of `shared` into 4 sub-keys
- [x] **A1.4** `Header::build(route, payload, rng) → [u8; HEADER_SIZE]`
- [x] **A1.5** `Header::unwrap_at_hop(hop_keys) → (next_routing, next_header)`
- [x] **A1.6** Poly1305 MAC chain over header (`γ` field)
- [x] **A1.7** Re-blinding of `(α, α*)` per-hop
- [x] **A1.8** Folded-KEM construction (commitment in header, CT in payload)
- [x] **A1.9** `GothamPacket::wrap(payload, route, rng) → [u8; PACKET_SIZE]` (Sphinx v0.2, fixed 2048-byte packets)
- [x] **A1.10** `GothamPacket::unwrap_at_relay(pkt, keys) → Action`
- [x] **A1.11** Padding strategy (zero-fill + length-hiding)
- [x] **A1.12** Unit tests: round-trip wrap/unwrap with 3-5 hops
- [x] **A1.13** Property tests with `proptest`: random payloads, random routes
- [ ] **A1.14** Fuzz harness with `cargo-fuzz`: unwrap on malformed packets must never panic or leak via timing *(planned)*
- [x] **A1.15** Benchmarks: wrap, unwrap, full 3-hop round-trip on local loopback
- [x] **A1.16** Per-hop payload protected by LIONESS wide-block non-malleable PRP (Anderson-Biham 4-round) — replaced the earlier XOR / per-hop-AEAD branch payload; defeats tagging
- [x] **A1.17** Reject low-order X25519 points (`was_contributory`) — fixes the old M-1 Sphinx low-order-points finding

**DoD Phase 1.** Round-trip works for 3-5 hops in unit tests, fuzz finds no panics in 1 h of run, wrap+unwrap < 1 ms on M-series Mac / Ryzen.

### Phase 2 — Relay binary (2 weeks)

- [x] **A2.1** `gotham-relay` standalone binary with CLI args (config, keys)
- [x] **A2.2** `ReplayCache` (5-min gamma-MAC replay cache, bounded size)
- [x] **A2.3** Poisson delay scheduler (Loopix-style, per-hop, configurable λ)
- [x] **A2.4** Stateless `Relay::process(packet) → Forward | Drop | DeliverLocal`
- [x] **A2.5** QUIC listener (`quinn`) on port 443 UDP
- [x] **A2.6** Noise XK per-link with rekey every 1 h
- [x] **A2.7** Graceful shutdown + key rotation hook
- [x] **A2.8** Prometheus metrics endpoint (counters only — no per-packet data, no IPs)
- [x] **A2.9** `systemd` service file + minimal-rights user account template
- [ ] **A2.10** Sandboxing (`seccomp-bpf` for syscall filtering + chroot) *(planned)*
- [x] **A2.11** Reproducible build (Cargo workspace `release` profile already pinned)

**DoD Phase 2.** Three relays running on three VPS in three countries; client can complete a round-trip; metrics show no PII; relay crash-restart doesn't lose any forwardable state (because there is none). *(Status: a directory authority + 3 relays are online, but all 3 currently sit on a single /16, so the global path-diversity guard correctly refuses to build a route — no real message has yet transited the live network. Not met until relays span multiple /16s.)*

### Phase 3 — Directory authority + path selection (2 weeks)

- [x] **A3.1** `RelayDescriptor` struct + serde for signed JSON
- [x] **A3.2** `Directory::verify(doc, authority_pubkey)` → `bool`
- [x] **A3.3** `gotham-directory` publisher (single Ed25519 sig)
- [x] **A3.4** Path-selection algorithm with GLOBAL diversity — distinct operator + distinct network (/16 for IPv4, /48 for IPv6) across the whole path, entry ≠ exit (§5.2)
- [x] **A3.5** Per-mode hop count + delay (`low-latency`, `balanced`, `paranoid`)
- [x] **A3.6** Local directory cache + last-known-good fallback
- [x] **A3.7** Anonymous directory refresh (fetch through Gotham itself)
- [x] **A3.8** Self-forming signed directory + directory authority (enroll / heartbeat / proof-of-possession liveness probe)
- [x] **A3.9** k-of-n authority attestation anchoring the decentralized P2P gossip transport (relays discover each other)

**DoD Phase 3.** Client picks valid paths from a signed directory; reject expired or wrong-signature directories; refresh works through the mixnet.

### Phase 4 — App integration (2 weeks)

- [x] **A4.1** Tauri command `gotham_send(payload, recipient_fp)` wraps + dispatches
- [x] **A4.2** Recipient-side receive loop subscribes to entry-relay socket
- [x] **A4.3** `send_message` routes over Gotham (sole transport since v0.7; Tor/Lokinet/SMP removed) — real user sends now flow through the cover-traffic queue
- [x] **A4.4** Migrate `set_own_profile` broadcast over Gotham
- [x] **A4.5** Gotham is the sole transport — the legacy transport toggle and its `tor-legacy` / `clearnet` options were removed in v0.7
- [x] **A4.6** Gotham mailbox for store-and-forward offline delivery — deposit done OVER the mixnet; recipients spread across mailbox hosts via HRW hashing
- [x] **A4.7** Migration UX: progress bar during first-time Gotham bootstrap
- [x] **A4.8** SURB (single-use reply blocks) enabling anonymous mailbox fetch

**DoD Phase 4.** A real DM message goes end-to-end via Gotham in < 300 ms median. *(Not yet met on the live network: no message has transited it — see Phase 2 status.)*

### Phase 5 — Cover traffic & metadata hygiene (1 week)

- [x] **A5.1** `CoverScheduler` with `λ` configurable per mode (continuous cover traffic; real user sends routed through the cover-traffic queue)
- [x] **A5.2** Drop packets (sink relay) + Loop packets (self-loop)
- [x] **A5.3** Indistinguishability of drop vs real at the wire level
- [ ] **A5.4** Battery-aware degradation (mobile) *(planned — mobile clients not shipped; desktop only)*
- [ ] **A5.5** Backgrounded-app pause + resume *(planned — mobile clients not shipped)*

**DoD Phase 5.** Wire observer cannot tell if a user is idle or actively chatting beyond a 30-second window. *(Holds once a live network spanning multiple /16s exists; today it is not yet real-world proven.)*

### Phase 6 — Pluggable transports (planned / not shipped)

> The real Gotham link layer today is Noise XK over QUIC (A6.1). The obfuscation /
> domain-fronting transports below are future work — none are shipped yet.

- [x] **A6.1** Default: QUIC over UDP/443 (Noise XK per-link)
- [ ] **A6.2** Fallback: TLS 1.3 over TCP/443 with realistic SNI *(planned / not shipped)*
- [ ] **A6.3** obfs4-like: random-looking bytes after handshake *(planned / not shipped)*
- [ ] **A6.4** meek-CDN: HTTPS to a domain-fronted CDN (CloudFront / Fastly) *(planned / not shipped)*
- [ ] **A6.5** Adaptive selection — try each in order at startup *(planned / not shipped)*
- [ ] **A6.6** Detection-evasion test suite against `nDPI`, `Suricata` *(planned / not shipped)*

**DoD Phase 6.** Client survives a deliberately blocked UDP/443 environment, falls back transparently, and the user notices only a 1-2 s extra startup. *(Future — pluggable transports not yet shipped.)*

### Phase 7 — Hardening & audit prep (2 weeks)

- [ ] **A7.1** Forbid `unwrap()` / `expect()` / `panic!` in crypto paths
  (already enforced via `#![deny(clippy::unwrap_used)]` in `lib.rs`)
- [ ] **A7.2** All crypto operations in constant time (verify with `dudect`)
- [ ] **A7.3** `zeroize` on every secret-bearing struct
- [ ] **A7.4** Formal verification of Sphinx unwrap (Kani or Creusot)
- [ ] **A7.5** Threat-model walkthrough against `crypto-gotham/src/lib.rs#THREAT_MODEL`
- [ ] **A7.6** Documentation: every public fn has a doc-comment with safety notes

**DoD Phase 7.** External reviewer can read the code top-down without needing to ask "what does this mean".

### Phase 8 — Mobile push relay (2 weeks)

> Future — mobile clients are not shipped (desktop only: Linux/macOS/Windows).

- [ ] **A8.1** `gotham-push-relay` separate binary
- [ ] **A8.2** Receives final-hop Gotham packets destined to push-enrolled users
- [ ] **A8.3** Encrypted push token (sealed to user's device key, opaque to relay)
- [ ] **A8.4** APNS + FCM integration with minimal payload
- [ ] **A8.5** Privacy-preserving token rotation (per-device, weekly)

**DoD Phase 8.** Receiving a Gotham message while the app is fully backgrounded on iOS/Android triggers a notification within 5 seconds.

**Total Track A: ~16 weeks** (single dev) or **~8-10 weeks** (two devs in parallel).

---

## TRACK B · Infrastructure & deployment

### Bootstrap (5 relays, ~40 €/month)

- [ ] **B1** Choose hosting providers — at least 3 distinct (e.g. Hetzner DE, OVH FR, Vultr SG, Linode JP, DigitalOcean US)
- [ ] **B2** Spin up 1 × entry, 3 × mix, 1 × exit
- [ ] **B3** Set up DNS for `relay-<n>.gotham.example` (Let's Encrypt)
- [ ] **B4** Deploy reproducible relay binaries (matching commit SHA)
- [ ] **B5** Verify mutual reachability of all relays
- [ ] **B6** Configure firewalls: only UDP/443 + TCP/443 + SSH from admin IPs
- [ ] **B7** Configure `fail2ban` for SSH brute-force prevention
- [ ] **B8** Set up centralized logging (relay only logs uptime + counters, never per-packet data)

### Directory deployment

- [ ] **B9** Static-file host for the signed directory JSON (Cloudflare Pages / Netlify)
- [ ] **B10** Daily refresh cron with new `valid_after`/`valid_until` window
- [ ] **B11** Directory authority Ed25519 key stored in YubiKey (offline signing)

### Monitoring (privacy-preserving only)

- [ ] **B12** Per-relay liveness probe (HTTP /health endpoint, no per-packet metrics)
- [ ] **B13** Aggregate uptime dashboard
- [ ] **B14** Alerts on > 5 min downtime (Telegram / Signal bot to LG)

### Scaling

- [ ] **B15** Documentation for community-run relays
- [ ] **B16** Onboarding workflow for new operators (key gen → vet → directory add)
- [ ] **B17** Bandwidth budget per relay tier
- [ ] **B18** Geographic diversity dashboard (countries / ASes represented)

**DoD Track B.** Five relays running publicly for ≥ 30 days with > 99% uptime, directory updated daily, no single point of failure.

---

## TRACK C · Security & cryptography review

- [x] **C1** Internal review against the threat model (§9 of `GOTHAM.md`) — internal audit 2026-05-25 plus several multi-agent adversarial reviews; all confirmed findings fixed
- [x] **C2** Static analysis: `cargo-audit` clean except 1 accepted risk — RSA-Marvin (RUSTSEC-2023-0071) via `openidconnect`, unreachable path, documented (`cargo-deny` / `cargo-vet` still to wire in)
- [ ] **C3** Dependency review — every transitive dep audited or pinned
- [ ] **C4** Constant-time verification (`dudect` or similar)
- [ ] **C5** Formal verification of critical paths (Sphinx unwrap, MAC chain)
- [ ] **C6** External audit engagement — NOT yet done (no external/third-party audit has taken place; only the internal audit + adversarial reviews under C1). Quote 3 firms:
  - [ ] Synacktiv (FR)
  - [ ] Quarkslab (FR)
  - [ ] NCC Group (UK / US)
  - [ ] Trail of Bits (US) — for international credibility
- [ ] **C7** Bug bounty program (HackerOne / YesWeHack) — at least 5 k€ pot
- [ ] **C8** Responsible-disclosure policy published
- [ ] **C9** CVE-numbering authority registration (mid-term)

**DoD Track C.** Independent audit produces a report with no Critical findings; all Mediums and below have an explicit remediation date.

---

## TRACK D · Certification & compliance (FR/UE enterprise target)

### CSPN (Certification de Sécurité de Premier Niveau, ANSSI)

- [ ] **D1** Identify scope (what part of Crypto is certified — likely the messaging + Gotham)
- [ ] **D2** Choose CSPN-approved evaluator (Quarkslab, Synacktiv, Amossys, Lexfo, Oppida)
- [ ] **D3** Prepare technical documentation pack (architecture, threat model, crypto rationale)
- [ ] **D4** Submit certification request to ANSSI
- [ ] **D5** Evaluation phase (~3-6 months)
- [ ] **D6** Address findings + re-eval if needed
- [ ] **D7** Receive CSPN — register product publicly

**Budget:** 50-80 k€ + ~12 months calendar.

### Common Criteria / Qualification Standard (later)

- [ ] **D8** Decide whether to pursue CC EAL2+ or ANSSI Qualification Standard
- [ ] **D9** If yes: scope, evaluator, budget (100-300 k€ + 18 months)

### RGPD / GDPR

- [ ] **D10** Data Protection Impact Assessment (DPIA)
- [ ] **D11** Privacy Policy + Terms of Service published in FR/EN
- [ ] **D12** Data Processing Agreement (DPA) template for enterprise customers
- [ ] **D13** Register of processing activities
- [ ] **D14** Designated DPO (Data Protection Officer) — can be a fractional role

### NIS2 / DORA readiness

- [ ] **D15** Map Gotham architecture against NIS2 article 21 risk-management measures
- [ ] **D16** Incident response plan (4-hour notification to ENISA-equivalent for "significant" incidents)
- [ ] **D17** Supply-chain security (SBOM, signed releases)

### SecNumCloud (if SaaS hosted offer)

- [ ] **D18** Evaluate whether to pursue SecNumCloud (relevant only for managed cloud offering)
- [ ] **D19** If yes: 12-18 months + 100-200 k€

### ISO 27001

- [ ] **D20** Define ISMS scope
- [ ] **D21** Risk assessment + treatment plan
- [ ] **D22** External audit (Bureau Veritas, AFNOR, BSI)

**DoD Track D.** CSPN obtained on Crypto + Gotham; CSPN logo visible on website; ready to respond to RFP from French defense industrials.

---

## TRACK E · Open-source & community

- [ ] **E1** Choose license (recommended: AGPLv3 for protocol implementation, MIT for client SDK)
- [ ] **E2** Publish `crypto-gotham` and `GOTHAM.md` on GitHub or self-hosted Gitea
- [ ] **E3** Code of Conduct (Contributor Covenant)
- [ ] **E4** CONTRIBUTING.md with security-disclosure section
- [ ] **E5** Maintainers list + governance doc
- [ ] **E6** Public mailing list / Matrix room for protocol discussion
- [ ] **E7** Initial blog post: "Introducing Gotham"
- [ ] **E8** Submit RFC for IETF informational track (long-term, for credibility)

**DoD Track E.** External developer can clone the repo, read `GOTHAM.md`, build a working relay, and join the network within an evening.

---

## TRACK F · Commercial enterprise (FR/UE defense target)

### Sales prep

- [ ] **F1** One-page pitch deck (problem, solution, differentiation vs Tchap/Olvid/Wire)
- [ ] **F2** Demo video (60-90 s) showing the user experience
- [ ] **F3** Technical white paper (~10 pages) for CISO/CTO audience
- [ ] **F4** Pricing model (per-user/month, on-prem license, support tiers)

### Channels

- [ ] **F5** Reference UGAP (Union des Groupements d'Achats Publics)
- [ ] **F6** Apply to ANSSI's "Visa de sécurité" (separate track from CSPN)
- [ ] **F7** Présence FIC (Forum International Cybersécurité, Lille — January)
- [ ] **F8** Présence Eurosatory (defense industry — June)
- [ ] **F9** Pilote ETI ou laboratoire (CNRS, INRIA, CEA) as reference

### Sales targets

- [ ] **F10** Tier-1: Airbus, Thalès, Dassault — formal RFP track
- [ ] **F11** Tier-2: Naval Group, MBDA, Safran, Nexter — secondary
- [ ] **F12** Tier-3: Gouvernement (after Tchap displacement) — long-term

**DoD Track F.** 1 paying enterprise customer signed; ARR ≥ 100 k€.

---

## TRACK G · Hardware / embedded technologies (long-term)

These are *future* directions for differentiation against software-only
competitors. None required for v1.

- [ ] **G1** Smart card support (PKCS#11) for long-term identity key storage
- [ ] **G2** YubiKey / SoloKey FIDO2 for second-factor unlock
- [ ] **G3** Secure Enclave (macOS / iOS) for ephemeral key generation
- [ ] **G4** Android StrongBox / TEE for key storage
- [ ] **G5** Hardware random-number sources (TRNG) for relay operators
- [ ] **G6** Air-gapped key ceremony for directory authority root key
- [ ] **G7** USB-C/Lightning hardware token v1 (mid-term R&D)

**Note.** G7 is a 12-18 month R&D project. Defer until first commercial traction.

---

## TRACK H · Documentation

- [ ] **H1** `GOTHAM.md` — Protocol specification (this directory)
- [ ] **H2** `GOTHAM-CHECKLIST.md` — This document
- [ ] **H3** `GOTHAM-OPSEC.md` — Operator guide (security best practices for relay admins)
- [ ] **H4** `GOTHAM-USERGUIDE.md` — End-user documentation
- [ ] **H5** `GOTHAM-THREAT-MODEL.md` — Extended threat model with attack trees
- [ ] **H6** `GOTHAM-DEPLOYMENT.md` — How to operate a relay
- [ ] **H7** `CHANGELOG.md` — Per-version changes
- [ ] **H8** API reference (`cargo doc --open` clean output, no warnings)

**DoD Track H.** A developer new to the project can become productive within one working day using only the docs.

---

## Cross-track dependencies

```
       A1 (Sphinx) ──┬──▶ A2 (Relay) ──▶ A3 (Directory) ──▶ A4 (Integration)
                     │                                      ▲
                     └──▶ A5 (Cover) ────────────────────────┘
                                                            
                                                  A6 (Transports) (parallel to A4-A5)
                                                  A7 (Hardening) (after A1-A6)
                                                  A8 (Push relay) (after A4)

Track B can start as soon as A2 has a runnable binary (~week 5).
Track C runs continuously from week 1.
Track D starts the CSPN application around week 14 (after A7 is well underway).
Track E can start any time; recommend public publication after A4 internal demo.
Track F starts when there's something demoable (after A4 + A5).
Track H starts week 1 (alongside A1) and continues throughout.
```

---

## Ritual

When a checkbox flips from `[ ]` to `[x]`, add a one-line commit message
referencing the item:

```
chore(checklist): A1.3 — constant-time HKDF sub-key expansion done
```

This makes the project's progress auditable and easy to grep.

---

*« L'eau persiste là où le Fremen la cache. L'épice afflue où le travail est fait. »*
