# Gotham — Master Checklist

> All tracks in one document. Tick `[x]` as items complete. Each section
> ends with a `Definition of Done` so we don't ship half-baked.

**Last updated:** 2026-05-23
**Project lead:** Angel
**Protocol design:** Angel

---

## TRACK A · Protocol implementation (Rust)

### Phase 1 — Sphinx packet (3 weeks)

- [ ] **A1.1** `hybrid::encapsulate(rng, x25519_pk, mlkem_pk) → (α, α', shared)`
- [ ] **A1.2** `hybrid::decapsulate(x25519_sk, mlkem_sk, α, α') → shared`
- [ ] **A1.3** Constant-time HKDF expansion of `shared` into 4 sub-keys
- [ ] **A1.4** `Header::build(route, payload, rng) → [u8; HEADER_SIZE]`
- [ ] **A1.5** `Header::unwrap_at_hop(hop_keys) → (next_routing, next_header)`
- [ ] **A1.6** Poly1305 MAC chain over header (`γ` field)
- [ ] **A1.7** Re-blinding of `(α, α*)` per-hop
- [ ] **A1.8** Folded-KEM construction (commitment in header, CT in payload)
- [ ] **A1.9** `GothamPacket::wrap(payload, route, rng) → [u8; PACKET_SIZE]`
- [ ] **A1.10** `GothamPacket::unwrap_at_relay(pkt, keys) → Action`
- [ ] **A1.11** Padding strategy (zero-fill + length-hiding)
- [ ] **A1.12** Unit tests: round-trip wrap/unwrap with 3-5 hops
- [ ] **A1.13** Property tests with `proptest`: random payloads, random routes
- [ ] **A1.14** Fuzz harness with `cargo-fuzz`: unwrap on malformed packets must never panic or leak via timing
- [ ] **A1.15** Benchmarks: wrap, unwrap, full 3-hop round-trip on local loopback

**DoD Phase 1.** Round-trip works for 3-5 hops in unit tests, fuzz finds no panics in 1 h of run, wrap+unwrap < 1 ms on M-series Mac / Ryzen.

### Phase 2 — Relay binary (2 weeks)

- [ ] **A2.1** `gotham-relay` standalone binary with CLI args (config, keys)
- [ ] **A2.2** `ReplayCache` (LRU + 5-min TTL, bounded size)
- [ ] **A2.3** Poisson delay scheduler (per-hop, configurable λ)
- [ ] **A2.4** Stateless `Relay::process(packet) → Forward | Drop | DeliverLocal`
- [ ] **A2.5** QUIC listener (`quinn`) on port 443 UDP
- [ ] **A2.6** Noise XK per-link with rekey every 1 h
- [ ] **A2.7** Graceful shutdown + key rotation hook
- [ ] **A2.8** Prometheus metrics endpoint (counters only — no per-packet data, no IPs)
- [ ] **A2.9** `systemd` service file + minimal-rights user account template
- [ ] **A2.10** Sandboxing (`seccomp-bpf` for syscall filtering + chroot)
- [ ] **A2.11** Reproducible build (Cargo workspace `release` profile already pinned)

**DoD Phase 2.** Three relays running on three VPS in three countries; client can complete a round-trip; metrics show no PII; relay crash-restart doesn't lose any forwardable state (because there is none).

### Phase 3 — Directory authority + path selection (2 weeks)

- [ ] **A3.1** `RelayDescriptor` struct + serde for signed JSON
- [ ] **A3.2** `Directory::verify(doc, authority_pubkey)` → `bool`
- [ ] **A3.3** `gotham-directory` static-file publisher (single Ed25519 sig)
- [ ] **A3.4** Path-selection algorithm with operator + AS diversity (§5.2)
- [ ] **A3.5** Per-mode hop count + delay (`low-latency`, `balanced`, `paranoid`)
- [ ] **A3.6** Local directory cache + last-known-good fallback
- [ ] **A3.7** Anonymous directory refresh (fetch through Gotham itself)
- [ ] **A3.8** N-of-M multi-sig directory (deferred to v2 — note for spec)

**DoD Phase 3.** Client picks valid paths from a signed directory; reject expired or wrong-signature directories; refresh works through the mixnet.

### Phase 4 — App integration (2 weeks)

- [ ] **A4.1** Tauri command `gotham_send(payload, recipient_fp)` wraps + dispatches
- [ ] **A4.2** Recipient-side receive loop subscribes to entry-relay socket
- [ ] **A4.3** Migrate `send_message` to call Gotham instead of SMP-over-Tor
- [ ] **A4.4** Migrate `set_own_profile` broadcast over Gotham
- [ ] **A4.5** Frontend setting: transport = `gotham` | `tor-legacy` | `clearnet`
- [ ] **A4.6** Backward-compat shim — accept inbound from old SMP for N months
- [ ] **A4.7** Migration UX: progress bar during first-time Gotham bootstrap

**DoD Phase 4.** A real DM message goes end-to-end via Gotham in < 300 ms median; old SMP path still functional during the transition window.

### Phase 5 — Cover traffic & metadata hygiene (1 week)

- [ ] **A5.1** `CoverScheduler` with `λ` configurable per mode
- [ ] **A5.2** Drop packets (sink relay) + Loop packets (self-loop)
- [ ] **A5.3** Indistinguishability of drop vs real at the wire level
- [ ] **A5.4** Battery-aware degradation (mobile)
- [ ] **A5.5** Backgrounded-app pause + resume

**DoD Phase 5.** Wire observer cannot tell if a user is idle or actively chatting beyond a 30-second window.

### Phase 6 — Pluggable transports (2 weeks)

- [ ] **A6.1** Default: QUIC over UDP/443
- [ ] **A6.2** Fallback: TLS 1.3 over TCP/443 with realistic SNI
- [ ] **A6.3** obfs4-like: random-looking bytes after handshake
- [ ] **A6.4** meek-CDN: HTTPS to a domain-fronted CDN (CloudFront / Fastly)
- [ ] **A6.5** Adaptive selection — try each in order at startup
- [ ] **A6.6** Detection-evasion test suite against `nDPI`, `Suricata`

**DoD Phase 6.** Client survives a deliberately blocked UDP/443 environment, falls back transparently, and the user notices only a 1-2 s extra startup.

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

- [ ] **C1** Internal review against the threat model (§9 of `GOTHAM.md`)
- [ ] **C2** Static analysis: `cargo-audit`, `cargo-deny`, `cargo-vet` clean
- [ ] **C3** Dependency review — every transitive dep audited or pinned
- [ ] **C4** Constant-time verification (`dudect` or similar)
- [ ] **C5** Formal verification of critical paths (Sphinx unwrap, MAC chain)
- [ ] **C6** External audit engagement — quote 3 firms:
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
