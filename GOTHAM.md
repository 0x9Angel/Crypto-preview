# Gotham Protocol — Specification v0.1

> **Status:** Draft — pre-alpha. Subject to breaking changes until v1.0.
> **Authors:** Angel
> **License:** TBD (recommended AGPLv3 + commercial exception)

---

## §0 · Table of contents

1. [Goals & non-goals](#1-goals-non-goals)
2. [Cryptographic primitives](#2-cryptographic-primitives)
3. [Packet wire format](#3-packet-wire-format)
4. [Cryptographic operations](#4-cryptographic-operations)
5. [Routing & path selection](#5-routing--path-selection)
6. [Relay protocol](#6-relay-protocol)
7. [Cover traffic](#7-cover-traffic)
8. [Directory authority](#8-directory-authority)
9. [Threat model](#9-threat-model)
10. [Conformance levels](#10-conformance-levels)
11. [Open questions](#11-open-questions)

---

## §1 · Goals & non-goals

### Goals

- **G1.** End-to-end content confidentiality preserved from sender to recipient.
- **G2.** Network-layer anonymity against any single relay (and any reasonable
  subset of relays not exceeding 50% of the pool).
- **G3.** Median round-trip latency ≤ 300 ms (low-latency mode ≤ 200 ms).
- **G4.** Resistance to replay, tagging, and active modification of in-flight
  packets.
- **G5.** Post-quantum hybrid security (X25519 + ML-KEM-768) from day 1.
- **G6.** Fixed-size packets so traffic analysis cannot infer message length.
- **G7.** Pluggable transports (QUIC default; TLS 1.3, obfs4-like and meek
  fallbacks) to survive aggressive DPI.
- **G8.** Stateless relays — beyond an in-memory replay cache, relays hold no
  persistent state that could be subpoenaed or seized.

### Non-goals

- **NG1.** Resistance to a Global Passive Adversary (NSA, 5-Eyes, …). No mixnet
  resists a true GPA; we explicitly do not claim to.
- **NG2.** Resistance to endpoint compromise. If the attacker controls the user
  device, the protocol cannot help.
- **NG3.** Resistance to forced key disclosure with non-ephemeral material.
  Ephemeral keys are zeroized after use; long-term identity keys can be
  protected only by the user (HSM / SecureEnclave is out of scope here).
- **NG4.** Anonymous relay registration. Relays are vetted by the directory
  authority for v1; web-of-trust admission deferred to v2.

---

## §2 · Cryptographic primitives

| Purpose                         | Primitive                          | Library            |
|---------------------------------|------------------------------------|--------------------|
| Classical KEX (per hop)         | X25519                             | `x25519-dalek`     |
| Post-quantum KEM (per hop)      | ML-KEM-768 (NIST FIPS 203)         | `ml-kem`           |
| AEAD (payload + headers)        | ChaCha20-Poly1305                  | `chacha20poly1305` |
| KDF                             | HKDF-SHA256                        | `hkdf`             |
| Hash (general)                  | BLAKE3                             | `blake3`           |
| MAC chain (header)              | Poly1305 (under HKDF subkey)       | `poly1305`         |
| Identity signatures (directory) | Ed25519                            | `ed25519-dalek`    |
| RNG                             | OS-backed; ChaCha20 for derivation | `rand`, `rand_chacha` |

All primitives have ≥ 5 years of public review. No bespoke crypto.

---

## §3 · Packet wire format

A Gotham packet is **exactly 2048 bytes** after construction.

```
┌──────────────────────────────────────────────────────────────────┐
│  Gotham Packet — 2048 bytes                                       │
├──────────────────────────────────┬───────────────────────────────┤
│  HEADER 384 bytes                │       PAYLOAD 1664 bytes      │
│                                  │                               │
│  α   (32 B)   X25519 ephem pk    │  Three-layer AEAD nesting:    │
│  α'  (1088 B fixed-but-folded)*  │  Enc_K1( Enc_K2( Enc_K3(      │
│  β   (≤ 256 B routing block)     │    SealedSender (             │
│  γ   (16 B Poly1305 MAC)         │      DoubleRatchet (msg)      │
│                                  │    ) ) ) )                    │
│  *ML-KEM-768 CT is 1088 B; we    │  + zero padding to fill 1664  │
│   compress via XOR-with-PRF      │                               │
│   (see §4.2) into the 320 B      │                               │
│   header budget. The full CT    │                               │
│   is referenced from a 32 B      │                               │
│   commitment α* in α'.           │                               │
└──────────────────────────────────┴───────────────────────────────┘
```

> **Note on the 384 B header.** A naive layout (α + α' + β + γ + per-hop
> records × MAX_HOPS) blows past 384 B because of ML-KEM's 1088 B ciphertext.
> Gotham uses a *folded KEM* construction: the full ML-KEM ciphertext lives
> at the start of the payload field (inside Enc_K1), while only a 32 B
> commitment to it sits in the header. This means the recipient of the
> outermost layer must verify the commitment opens the same ciphertext used
> for its own decapsulation — preventing substitution attacks. See §4.2.

### 3.1 Header sub-fields

```
struct Header {
    version: u8,                  // 0x01 for this spec
    mode: u8,                     // 0=low-latency, 1=balanced, 2=paranoid
    reserved: u16,                // must be zero
    alpha: [u8; 32],              // X25519 ephemeral public key
    alpha_star: [u8; 32],         // commitment to ML-KEM ciphertext
    alpha_prime_blob: [u8; ...],  // folded ML-KEM material — see §4.2
    beta: [u8; β_LEN],            // encrypted routing info
    gamma: [u8; 16],              // Poly1305 MAC over (α, α*, α', β)
}
```

`β_LEN` is computed such that total header = 384 bytes for the chosen
`MAX_HOPS = 5`. Each hop consumes a fixed slice of β.

### 3.2 Routing record (one per hop, encrypted in β)

```
struct RoutingRecord {
    next_addr: [u8; 18],          // 16-byte IPv6 + 2-byte port (IPv4-mapped if needed)
    next_node_id: [u8; 32],       // Ed25519 fingerprint of next relay
    delay_micros: u32,            // Poisson delay parameter for this hop
    flag: u8,                     // bit0: is_last_hop; bit1: deliver_local
    padding: [u8; 9],
}
```

Total per-record: 64 B. With MAX_HOPS = 5 and per-hop key wraps included,
β_LEN ≈ 256 B.

---

## §4 · Cryptographic operations

### 4.1 Hybrid shared secret

For each hop *i*, the relay decapsulates both:

```
ss_x  = X25519_DH(relay_x25519_sk, alpha_i)
ss_pq = ML-KEM-768.Decapsulate(relay_mlkem_sk, alpha_prime_i)
ss_i  = HKDF-SHA256(ss_x || ss_pq, info = "gotham-hop-secret-v1")
```

The HKDF then expands `ss_i` into four 32-byte sub-keys:

| Sub-key       | Use                                                  |
|---------------|------------------------------------------------------|
| `k_mac`       | Verify γ and produce next γ                          |
| `k_header`    | Decrypt β to extract this hop's routing record      |
| `k_payload`   | Strip the outermost AEAD layer of the payload       |
| `k_blind`     | Re-randomize (α, α*) for the next hop               |

### 4.2 Folded KEM construction

ML-KEM-768 ciphertexts are 1088 B — too large for the 384 B header. Gotham
inlines them at the head of the payload as:

```
payload[0..1088]   = ml_kem_ciphertext_for_hop_1
payload[1088..]    = Enc_K1( payload_for_next_hop )
```

The header carries `α* = SHA-256(ml_kem_ciphertext_for_hop_1)` to commit to
the value. The recipient verifies the commitment after extracting the
ciphertext from the payload prefix. This adds 32 B per packet but lets us
preserve fixed packet size and avoid header bloat.

### 4.3 MAC chain (anti-tagging)

The header MAC `γ` is computed as:

```
γ = Poly1305(k_mac, version || mode || α || α* || folded_α' || β)
```

Any in-flight modification at hop *j* by an attacker controlling hop
*j−1* invalidates the MAC at hop *j* and the packet is dropped silently.

### 4.4 Re-blinding

After unwrap, the relay produces the next-hop header by:

```
α_{i+1}      = X25519(k_blind, α_i)       // scalar-mult re-randomization
α*_{i+1}     = drawn from PRF(k_blind || "alpha_star")
```

The peeled β becomes β_{i+1}, padded with random bytes at the tail to
maintain fixed length.

---

## §5 · Routing & path selection

### 5.1 Modes

| Mode          | Hops | Per-hop μ (Poisson delay) | Target latency |
|---------------|------|--------------------------|----------------|
| `low-latency` | 3    | 10 ms                    | 100-200 ms     |
| `balanced`    | 4    | 20 ms                    | 200-300 ms     |
| `paranoid`    | 5    | 50 ms                    | 350-600 ms     |

### 5.2 Selection constraints

When choosing a path:

1. **Tier discipline.** First hop = `entry`, last hop = `exit`, intermediate
   hops = `mix`. Mismatch is a path-selection error.
2. **Operator diversity.** No two consecutive hops from the same operator
   (when operator metadata is published in the directory).
3. **AS diversity.** No two consecutive hops in the same /16 IPv4 or /48
   IPv6 prefix.
4. **Country diversity (best effort).** Prefer paths spanning at least 2
   distinct countries.

### 5.3 Per-packet randomization

In `paranoid` mode, the client re-randomizes the full path on every
packet. In `balanced` / `low-latency`, the same path is reused for up to
10 minutes or 100 packets — whichever is shorter — then rotated.

---

## §6 · Relay protocol

### 6.1 State machine

```
        ┌──────────────────────────────────────────────┐
        ▼                                              │
   [LISTENING] ──packet arrives──▶ [VERIFY_MAC] ──ok──▶│
                                       │               │
                                       └─fail─▶ [DROP]│
                                                       │
   [VERIFY_MAC ok] ──▶ [CHECK_REPLAY] ──hit─▶ [DROP]──┤
                                                       │
   [CHECK_REPLAY ok] ──▶ [DECAP_HOP_KEYS] ──fail─▶ [DROP]
                                  │                    │
                                  ▼                    │
                          [UNWRAP_HEADER]              │
                                  │                    │
                                  ▼                    │
                          [POISSON_DELAY]              │
                                  │                    │
                                  ▼                    │
                     ┌─is_last_hop?─┐                  │
                     │              │                  │
                    yes             no                 │
                     │              │                  │
                     ▼              ▼                  │
              [DELIVER_LOCAL]  [STRIP_PAYLOAD_LAYER]   │
                                   │                   │
                                   ▼                   │
                            [REBLIND_HEADER]           │
                                   │                   │
                                   ▼                   │
                         [FORWARD_NEXT_HOP] ───────────┘
```

### 6.2 Replay cache

In-memory `LRU<[u8; 16], Instant>` keyed by `γ`. TTL = 5 minutes. Bounded
size (default 1 M entries). On overflow, oldest entries evicted.

Hot-path requirement: each packet incurs exactly one cache lookup + one
optional insert. No disk persistence — the cache is rebuilt from zero on
relay restart (which is fine: an adversary can only replay packets newer
than the restart, and our keys rotate hourly anyway).

### 6.3 Link layer

Between every relay pair (and between client ↔ entry-relay), traffic is
wrapped in **Noise XK over QUIC** (or TCP+TLS 1.3 as fallback).

- Noise XK: relay identity is the static key; client/peer is anonymous.
- Rekey every 1 hour or 1 GiB of data.
- 0-RTT QUIC resumption permitted within the same session window.

---

## §7 · Cover traffic

Every Gotham client maintains a Poisson process with rate `λ`:

```
λ_low-latency = 1/15  // packet every 15 s on average
λ_balanced    = 1/10  // every 10 s
λ_paranoid    = 1/5   // every 5 s
```

At each tick:

- If the client has a real message to send → send it.
- Otherwise, send a **drop packet** (destination = special sink relay; the
  sink silently discards) **or** a **loop packet** (destination = self; the
  packet traverses the mixnet and arrives back as a "you sent yourself a
  message" event the app drops).

The two cover-traffic types are equiprobable to defeat statistical
discrimination.

Battery-aware degradation (mobile):

- If battery < 30% **and** charger disconnected → `λ` divided by 4 (less
  cover, more anonymity exposure, but device survives).
- If app is backgrounded for > 10 min → cover paused; resumed on app
  return.

---

## §8 · Directory authority

### 8.1 Document shape

```json
{
  "version": 1,
  "valid_after":  1700000000,
  "valid_until":  1700086400,
  "relays": [
    {
      "id":         "ed25519:abc…",
      "kem":        "mlkem768:def…",
      "x25519":     "x25519:ghi…",
      "addr":       "[2001:db8::1]:443",
      "tier":       "entry" | "mix" | "exit",
      "country":    "FR",
      "asn":        12876,
      "operator":   "Angel",
      "uptime_pct": 99.7
    }
  ],
  "authority": "ed25519:directory_auth_pubkey",
  "signature": "ed25519:base64sig"
}
```

### 8.2 Refresh

Clients fetch the directory **through the mixnet itself** (eat-own-dogfood)
from a designated `directory_responder` relay. Cached locally for 24 h.
On expiry, refresh; on fetch failure, fall back to last-known-good for up
to 7 days before refusing to operate.

### 8.3 Multi-sig (v2)

Initially the authority is a single Ed25519 keypair held by the project
maintainer. v2 introduces N-of-M multi-sig (default 3-of-5) with rotating
quorum members, mirroring Tor's directory consensus model but simplified.

---

## §9 · Threat model

See `crypto-gotham/src/lib.rs#THREAT_MODEL` for the canonical summary.
Detailed analysis below.

### 9.1 Adversaries Gotham resists

| Adversary                                | Mechanism                                                                |
|------------------------------------------|--------------------------------------------------------------------------|
| Passive ISP / café WiFi                  | Link encryption (Noise XK over QUIC/TLS); pluggable transports for DPI   |
| Single compromised relay                 | Sealed-sender + sphinx-onion: relay sees neither src nor dst nor content |
| Replay attempts                          | 5-min γ cache (§6.2)                                                     |
| Tagging attacks                          | MAC chain in header (§4.3)                                               |
| Timing correlation (small N)             | Poisson delays + cover traffic                                           |
| Packet-size analysis                     | Fixed 2 KB packets                                                       |
| Future quantum adversary                 | X25519 + ML-KEM-768 hybrid; an attacker must break both                  |
| Forced key disclosure (subpoena)         | Forward secrecy + ephemeral per-hop keys; nothing persistent to disclose |

### 9.2 Adversaries Gotham does NOT resist

| Adversary                                | Why                                                                       |
|------------------------------------------|---------------------------------------------------------------------------|
| Global Passive Adversary (NSA, 5-Eyes)   | Cover traffic mitigates but cannot eliminate end-to-end correlation       |
| > 60% relay compromise                   | Sampling probabilistically catches client; matches Tor's failure mode     |
| Endpoint malware                         | Out of scope — addressed by app-layer hardening (sandbox, SQLCipher)     |
| Physical hardware attacks (EM, fault)    | Out of scope                                                              |
| Side-channel CPU timing                  | Mitigated by constant-time primitive impls but not formally verified yet |

---

## §10 · Conformance levels

A Gotham implementation may claim conformance at one of three levels:

- **Gotham-MUSTS** — implements §3, §4, §6.1, §6.2, §6.3, §8.
- **Gotham-SHOULDS** — adds §5.3 (per-packet randomization), §7 (cover),
  §6.3 rekey.
- **Gotham-MAY** — adds §5 paranoid mode, multi-sig directory (§8.3), and
  formally-verified Sphinx unwrap.

CSPN-target implementations must reach Gotham-SHOULDS minimum.

---

## §11 · Open questions

- **OQ1.** Should we ship traffic shaping (constant-rate, like Vuvuzela)
  rather than Poisson? Trade-off: better anonymity vs higher bandwidth cost.
- **OQ2.** Multi-device support: same identity, multiple Gotham instances.
  Need protocol for state sync without leaking metadata.
- **OQ3.** Group messaging routing — naive fan-out blows up cover budget.
  Likely needs separate spec.
- **OQ4.** Push notifications: the push-relai needs minimal metadata about
  the recipient. How minimal can we make this without breaking iOS/Android
  background semantics?
- **OQ5.** Audit cadence: every minor version, every year, or event-driven
  (post-incident)?

---

*End of v0.1. Track changes in `CHANGELOG.md`.*
