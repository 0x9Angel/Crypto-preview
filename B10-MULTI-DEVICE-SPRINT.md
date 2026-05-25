# B10 — Multi-Device Sync · Sprint Plan

**Status:** scheduled (post-seed close, Q2-Q3 2027 window).
**Estimated duration:** ~4 weeks calendar · 1 design + 2 implementation + 1 review.
**Estimated budget:** dev time + ~15 k€ third-party crypto audit gate.
**Lead:** TBD (Project Lead until a cryptographer-in-residence is hired).
**Last reviewed:** 2026-05-25.

---

## 1 · Why this is a sprint, not a session

B10 is the single Track-B item that cannot safely ship inside a coding
session. The reason is not implementation effort — it is **protocol
design risk**. A naive multi-device implementation introduces a
catalogue of well-documented failure modes:

- **Ratchet state forks** when two devices decrypt the same message
  concurrently → out-of-order decryption + permanent loss of forward
  secrecy.
- **Identity-key exfiltration** during pairing — a flawed PAKE allows
  a near-attacker to impersonate the joining device.
- **Cross-device replay** — a message delivered to device A but not
  yet replicated to device B can be silently dropped on B's
  reconciliation.
- **Increased blast radius** — a second active device doubles the
  steal-the-master-password attack surface.
- **Operational asymmetry** — desktop and mobile have different
  lifetime, network-availability, and revocation profiles.

The right defence is not "ship faster". It is "design carefully,
review externally, then ship". This plan reflects that.

---

## 2 · Sprint goals (definition of done)

1. A user signs in on device A, scans a QR or types a 6-digit code on
   device B, and B is now a participating device on the account.
2. A message received on A also appears on B within ~30 s, decrypted
   and displayed.
3. A message *sent* on A also appears on B in the chat log (own
   outbound replication).
4. A device can be unlinked from any other device, immediately
   removing its ability to decrypt new messages.
5. The cryptographic design has passed independent review by an
   external firm engaged for the Trail-of-Bits / Quarkslab / Synacktiv
   audit (see [GOTHAM-CHECKLIST.md](GOTHAM-CHECKLIST.md) Track C).
6. Tests cover: pairing happy path · pairing under wrong code ·
   concurrent message receive · device unlink · forward-secrecy
   sanity (no plaintext leak when an unlinked device is later
   compromised).

---

## 3 · Phase 1 — Protocol design (1 working week)

### 3.1 Decisions to commit before any code is written

| # | Decision | Options to evaluate |
|---|---|---|
| **D1** | Pairing primitive | SPAKE2 (recommended) · CPace · simple short-code over Gotham-encrypted channel |
| **D2** | Device-key sharing | Full identity-key clone (Signal default) vs. per-device subkeys derived from a shared root (more modern) |
| **D3** | Ratchet replication granularity | Per-message events vs. periodic snapshot vs. operational-transform-style merge |
| **D4** | Authority-of-truth | "Primary device" elected at link-time · OR per-conversation primary · OR no primary (CRDT-style merge) |
| **D5** | Pairing code lifetime | 60s · 5min · 24h. Trade-off: shorter = better OPSEC, harder UX on slow mobile |
| **D6** | Maximum devices per account | 3 · 5 · unbounded. Trade-off: more devices = larger attack surface and slower reconciliation |
| **D7** | Cross-device sync transport | Reuse Gotham mixnet · vs. a dedicated relay · vs. direct device-to-device over QUIC |

### 3.2 Deliverables (Phase 1 end)

- `GOTHAM-MULTI-DEVICE-PROTOCOL.md` at repo root — 15-25 page spec
  documenting D1-D7 with rationale + the chosen handshake + the
  replication state machine.
- Updated `GOTHAM-THREAT-MODEL.md` §3 with two new threat classes:
  multi-device-state-fork and device-pairing-side-channel.
- Sequence diagrams for: pair-new-device · receive-on-A-replicate-to-B
  · send-on-B-coordinate-with-A · unlink-device.
- A formal model in TLA+ or Tamarin for the pairing handshake (the
  ratchet replication is easier to argue informally; the pairing
  half-message exchange is where formal verification earns its keep).

### 3.3 Exit criteria for Phase 1

- The design has been reviewed by the Project Lead **plus at least
  one external cryptographer** (an advisor, not yet the audit firm).
- All decisions D1-D7 are committed in writing with rationale.
- No question on the sequence diagrams returns "I don't know" from
  the design owner.

---

## 4 · Phase 2 — Implementation (10-12 working days)

### 4.1 Crate-level work plan

| Crate | New / modified | Effort |
|---|---|---|
| `crypto-proto` | New wire types: `PairingInit`, `PairingResponse`, `SyncEvent`, `DeviceUnlink`. New message-type tag for sync-channel envelopes. | 1-2d |
| `crypto-agent` | Adapt `SessionManager` to share ratchet state via the chosen replication scheme. Add `apply_sync_event(ev)`. | 3-4d |
| `crypto-store` | Schema migration v18: extend `device_registry` (already shipped as v17 scaffold) with `peer_pk_x25519`, `link_status`, `linked_at`. New `device_sync_log` table for in-flight events. | 1d |
| `crypto-gotham-relay` | Sync channel routing — a paired peer is reached by its `peer_pk_x25519` on the existing Gotham mixnet (reuse send_sealed). | 1d |
| `crypto-tauri` (backend) | `pair_device_initiate()` returns short-code + QR. `pair_device_accept(code)` runs SPAKE2 + identity-clone + initial ratchet sync. `unlink_device(device_id)` revokes a peer. Inbound sync events from the Gotham drainer dispatched to `crypto-agent`. | 3-4d |
| `crypto-tauri` (frontend) | Settings → Devices → Link a device · QR display · accept-pairing flow on the second device · unlink confirmation with re-auth gate. | 2-3d |

Total: **~11-15 working days**. Round up to 2 calendar weeks at 1 dev.

### 4.2 Coding standards (carried over from the rest of the project)

- `#![cfg_attr(not(test), deny(clippy::unwrap_used))]` on every new
  cryptographic path.
- `zeroize` on every secret-bearing struct.
- No `panic!` in production paths.
- Every public fn has a doc comment with safety notes.
- Every cross-crate seam has at least one integration test.

### 4.3 Test coverage targets (Phase 2 end)

| Path | Test type | Coverage goal |
|---|---|---|
| SPAKE2 handshake | unit | round-trip + wrong-code reject + replay reject |
| Ratchet replication | unit + property-based | 100 concurrent receivers on the same chain must converge |
| Device pairing | integration | end-to-end pair + first message on both devices |
| Unlink | integration | new message on A is undecryptable on the unlinked B |
| Forward secrecy | integration | compromising a device today leaks no messages from yesterday |

---

## 5 · Phase 3 — Testing & hardening (1 working week)

### 5.1 Fuzz

- New fuzz targets:
  - `fuzz_pairing_init_decode`
  - `fuzz_pairing_response_decode`
  - `fuzz_sync_event_apply`

Run 24 h continuous on each target before declaring Phase 3 complete.

### 5.2 Formal verification

If Phase 1 produced a Tamarin/TLA+ model of the pairing handshake,
mechanically check the proofs of:

- **Mutual authentication**: at the end of handshake both peers agree
  on each other's identity.
- **Forward secrecy**: compromise of either device's long-term key
  after handshake does not reveal the established session key.
- **Key separation**: the pairing-derived key cannot be replayed to
  decrypt regular Gotham traffic.

### 5.3 Side-channel review

- Run `dudect`-style timing benches on the pairing key-comparison path.
- Verify constant-time SPAKE2 implementation (either the upstream
  crate or our own port).

### 5.4 Manual operational testing

- Pair desktop ↔ desktop on the same LAN.
- Pair desktop ↔ Android over the mixnet.
- Pair desktop ↔ iPhone over the mixnet.
- Unlink one of the three, confirm both surviving devices keep
  working and the unlinked one loses access on its next poll.
- Test pairing-code reuse → must be rejected.
- Test pairing-code expiry → must be rejected after the configured
  TTL.

---

## 6 · Phase 4 — External audit gate (3-4 weeks calendar, parallelisable with Phase 5)

This is the **non-negotiable gate** before merging the multi-device
branch into `main`.

### 6.1 Scope of audit

- The PAIRING handshake (D1) + IDENTITY SHARING scheme (D2).
- The REPLICATION protocol (D3) and conflict resolution (D4).
- The UNLINK flow + post-unlink key hygiene.

NOT in scope (already audited / out of this sprint):

- The underlying Gotham mixnet (separately audited).
- The Signal-style E2E layer (Signal protocol — audited).

### 6.2 Engagement model

Quote requested from at least three firms:

- **Trail of Bits** (US) — strongest for protocol-design review.
- **Quarkslab** (FR) — strongest for sovereign-EU + ANSSI overlap.
- **Synacktiv** (FR) — strongest for application-side integration.

Expected cost: **15-30 k€** for the multi-device scope alone, or
folded into the larger Gotham audit (see CHECKLIST.md Track C).

### 6.3 Audit deliverables

- A written report with classified findings (Critical / High /
  Medium / Low / Informational).
- A remediation matrix mapping each finding to a tracked work item.
- A re-test pass after remediation.

### 6.4 Exit criteria

- **Zero Critical findings.**
- All High findings remediated.
- Medium findings have an explicit remediation date.
- A published advisory if anything Critical or High required a
  protocol change.

---

## 7 · Phase 5 — Merge + ship (1 working week)

- Merge the multi-device branch into `main` (currently expected to be
  `b10-multi-device` or similar).
- Bump `PROTOCOL_VERSION` if any wire-format change crosses a hop.
- Publish a release-notes blog post explaining the security
  trade-offs to end users.
- Update `GOTHAM-USERGUIDE.md` with a "Linking another device" section.
- Add a "Devices" entry to the OnboardingWizard so first-launch users
  know the feature exists.

---

## 8 · Prerequisites before this sprint can start

| # | Prerequisite | Status today |
|---|---|---|
| **P1** | Seed funding closed (covers audit budget) | ❌ targeted Q1 2027 |
| **P2** | Co-founder / cryptographer-in-residence on board | ❌ |
| **P3** | One external cryptographer available for Phase 1 review | ❌ TBD |
| **P4** | Audit firm engagement signed (Phase 4) | ❌ TBD |
| **P5** | Existing single-device codebase is audit-clean (P5 depends on the upstream Gotham audit) | ❌ scheduled Q2-Q3 2027 |
| **P6** | Device-registry scaffold landed (V17 schema + UI) | ✅ shipped 2026-05-25 (commit `22de1ac`) |

The earliest realistic start of this sprint is therefore **after the
Gotham audit completes** — roughly Q3-Q4 2027 in the master roadmap.

---

## 9 · Open questions to resolve in Phase 1

These are deliberately left open because they belong to the design
phase, not this plan:

1. **Should we ship "primary device" semantics or full peer-to-peer
   CRDT-style sync?** The former is simpler to implement and audit;
   the latter is more resilient to a stolen primary device.
2. **Per-conversation device opt-in?** Some users may want a desktop
   to participate only in some chats, not all.
3. **What is the user-facing recovery story when a device is lost
   without prior unlink?** The remaining devices need to be able to
   revoke the lost device's keys (key-rotation broadcast).
4. **Backup interplay with B2 recovery code.** When a recovery code
   is used to reset the master password, do the linked devices
   re-pair, lose pairing, or silently keep working?
5. **Mobile-specific lifetime quirks.** iOS app-suspend → no background
   ratchet sync → device falls behind. Acceptable lag bound?

---

## 10 · Risks the sprint must surface in Phase 1

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Ratchet state fork on concurrent receive | Medium | High (FS loss) | Per-conversation primary device OR mandatory snapshot frequency |
| Pairing code phished | Medium | High (account theft) | Short TTL + single-use + same-network preference |
| Audit finds a Critical → 6-12 month delay | Medium | Very high (schedule slip) | Run formal verification in Phase 3, not Phase 4 |
| Mobile background limitations break sync | High | Medium (UX degradation) | Push-notification driven catch-up on app resume |
| Device-registry schema needs another migration | Medium | Low | V18 already anticipated in this plan |

---

## 11 · Out of scope for this sprint

To keep the sprint bounded, the following are explicitly EXCLUDED
and tracked as separate items:

- **Sender Keys ratchet for group chats** (currently in B3 hardening
  backlog — orthogonal to multi-device).
- **Push-notification fan-out across devices** (depends on A4/A5
  hot paths plus this sprint's replication channel — addressed once
  both are merged).
- **Device-side analytics / device-management dashboard** — pure
  enterprise feature for v2.
- **Cross-account device sharing** — explicitly forbidden by design;
  devices belong to exactly one account.

---

## 12 · Success criteria summary

The sprint is "done" when:

1. ✅ A user can link a second device end-to-end.
2. ✅ Messages received on either device appear on both.
3. ✅ Devices can be unlinked with immediate effect.
4. ✅ External audit produces a report with zero Critical findings.
5. ✅ All sprint tests are green; coverage on the new paths ≥ 85 %.
6. ✅ `GOTHAM-MULTI-DEVICE-PROTOCOL.md` is published in the repo.
7. ✅ The release-notes blog post is drafted (not necessarily
   published — publication waits for the overall enterprise GA
   window).

After this sprint, B10 in [CHECKLIST.md](CHECKLIST.md) flips from
`[~]` (scaffold) to `[x]` (shipped + audited).
