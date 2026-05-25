# Gotham — Extended Threat Model

This document extends the inline summary in
[crypto-gotham/src/lib.rs](crypto-gotham/src/lib.rs#L80) (`THREAT_MODEL`)
and the protocol specification in [GOTHAM.md](GOTHAM.md).

Its purpose is to make explicit, for each class of risk, the design
property that defends against it and the residual surface that
remains. It is a defensive document. It does not provide guidance
on how to attack a real deployment; it explains what the network
is built to withstand and what is out of scope.

**Audience:** auditors, integrators, researchers, RSSI / CISO,
sceptical end users.

**Last reviewed:** 2026-05-25

---

## 1. Scope

This threat model covers:

- The Gotham mixnet protocol (Sphinx-style packet wrap/unwrap,
  hybrid X25519+ML-KEM-768 key encapsulation, MAC chain, fixed-size
  packets, cover traffic, replay cache).
- The relay binary (`crypto-gotham-relay`) and its public surface
  (QUIC over UDP/443, Noise XK at the link layer).
- The client integration in Crypto (`crypto-tauri`'s `gotham_*`
  commands, the inbox drainer, the directory cache).
- The directory authority (signed JSON document, Ed25519 signature).

It does not cover:

- The end-to-end encryption layer (Signal protocol — X3DH + Double
  Ratchet). That layer is independently audited; this document
  assumes it is intact.
- The application's local storage (SQLCipher + Argon2id). See the
  Crypto general threat model for that.
- The endpoint operating system or hardware. Endpoint compromise
  is explicitly out of scope.

---

## 2. Security goals

In priority order:

| # | Goal | Defended by |
|---|------|-------------|
| **G1** | An external observer cannot determine *who is talking to whom*. | Mixnet routing, fixed-size packets, cover traffic, sealed-sender envelope. |
| **G2** | An external observer cannot determine *when a user is active*. | Cover traffic indistinguishable from real traffic, Poisson delay scheduling. |
| **G3** | A single compromised relay learns nothing about the end-to-end relationship. | Sphinx layered encryption, MAC chain, re-blinding per hop. |
| **G4** | Replay of an old packet cannot be used to extract information. | 5-minute MAC cache per relay, packet-level uniqueness. |
| **G5** | A future quantum adversary cannot retroactively decrypt today's traffic. | Hybrid X25519 + ML-KEM-768 KEM. |
| **G6** | Tampering with a packet in flight is detected and the packet dropped. | Sphinx MAC chain, AEAD on each per-hop record. |
| **G7** | The wire is indistinguishable from generic encrypted traffic at the IP layer. | Fixed-size 2048-byte packets, QUIC on UDP/443. |

Goals G1-G7 are met against the threat classes enumerated in §3.

---

## 3. Threat classes

We enumerate adversary capabilities, then map each to the defence
properties.

### 3.1 Passive observers (ISP, café Wi-Fi, network tap)

**Capability:** can read all packets on a link the user crosses.
Cannot inject or modify.

**Defence:**

- Fixed-size 2048-byte packets eliminate length-based correlation.
- QUIC + TLS hides the application protocol from any observer that
  is not also breaking TLS.
- Cover traffic produces a continuous background stream so periods
  of inactivity are invisible.
- The Sphinx wrap means the observer sees only opaque ciphertext
  per packet, with no plaintext routing information.

**Residual surface:**

- The observer can see *that* you are running a Gotham client (the
  pattern of fixed-size packets on UDP/443 is recognisable with
  effort). They cannot see *what* or *when*.

### 3.2 Single compromised relay (any tier)

**Capability:** a relay operator has been bribed, hacked, or coerced.
Can observe everything passing through that relay.

**Defence:**

- Sphinx layered encryption: the relay learns only the next hop and
  the routing record for itself. The full path is not derivable.
- Re-blinding: the `α` (alpha) public key is freshly re-randomised
  per hop, so two relays handling the same packet cannot trivially
  correlate it.
- MAC chain: the relay cannot tamper with a downstream record
  without breaking the next hop's MAC verification.
- Sealed-sender envelope at the payload layer: even the *last* hop,
  which delivers locally, does not learn who the sender was.

**Residual surface:**

- The first relay (entry) learns the client's IP address (it has to,
  to receive the packet). It does not learn the destination, the
  payload, or the user identity inside the sealed envelope.
- The last relay (exit) learns the recipient's address. It does not
  learn the sender or the payload.

### 3.3 Single tampering relay (active malicious behaviour)

**Capability:** a relay tries to modify packets in transit, drop
selectively, or inject crafted packets.

**Defence:**

- Tampering: any modification to a routing record or header content
  invalidates the next hop's MAC. The next relay drops the packet.
  No information leaks.
- Selective drop: a relay can drop any packet it sees, but cannot
  identify which packets matter to which conversation (Sphinx).
  Random drop attacks degrade availability without revealing
  metadata.
- Injection: a relay can inject crafted packets, but they will
  appear in the wire as any other packet. They cannot impersonate
  another relay's signature (the MAC chain is keyed per hop).

**Residual surface:**

- Availability degradation: a malicious relay can refuse to forward
  honestly. The path selector retries through fresh relays. A
  pattern of refusal localised to one operator should be
  detectable in aggregate and warrants directory-level review.

### 3.4 Replay attempts

**Capability:** observe a packet, store it, re-inject it later
hoping to learn something from the relay's response.

**Defence:**

- Every relay maintains a 5-minute MAC cache. Replays within the
  TTL window are dropped silently.
- After the TTL expires, the hop keys derived from the packet's
  ephemeral material have rotated through the rest of the network,
  so re-injection has no useful effect.

**Residual surface:**

- A replay outside the 5-minute window will not produce a useful
  signal but consumes a small amount of relay resources for the
  failed validation. The replay cache is bounded to prevent
  resource exhaustion.

### 3.5 Tagging attacks

**Capability:** an adversary controls two relays on a path (e.g.
the entry and exit). Tries to mark a packet at the entry such that
the exit can recognise it later.

**Defence:**

- The MAC chain binds every byte of the header. Any modification at
  the entry invalidates the MAC at the second hop and the packet is
  dropped before it ever reaches the exit.
- Per-hop re-blinding of `α` means the on-wire representation of
  the packet between hops is mathematically distinct.

**Residual surface:**

- If the adversary controls more than 60% of the relay pool, they
  may achieve a path that is fully adversarial and apply
  out-of-band correlation. See §3.10 for the majority-compromise
  case.

### 3.6 Timing correlation at sub-state scale

**Capability:** observer sees both endpoints (entry-side traffic
and exit-side traffic) and tries to correlate by timing patterns.

**Defence:**

- Poisson-distributed per-hop delays randomise the time at which a
  packet emerges from a relay relative to when it entered.
- Cover traffic adds noise: the observer cannot tell whether any
  given packet is a real message or a dummy.
- Loop packets occasionally make a sender appear to send to
  themselves through a full path, further muddying the conversation
  graph.

**Residual surface:**

- For an observer with visibility into a *majority* of the network
  (a "global passive adversary"), Poisson noise and cover traffic
  do not suffice. See §3.10. This residual is documented as
  out-of-scope.

### 3.7 Wire-format detection (DPI, traffic classifiers)

**Capability:** an ISP or middlebox runs deep-packet inspection to
identify Gotham traffic and either log or block it.

**Defence:**

- All Gotham packets are 2048 bytes and travel over QUIC/UDP/443
  with realistic TLS-like flow patterns. Cover traffic ensures the
  flow continues even when the user is idle.
- The pluggable-transports framework (v0.2 feature branch) adds
  obfs4-like obfuscation and meek-CDN domain-fronting variants for
  hostile network environments.

**Residual surface:**

- A sufficiently determined classifier with a Gotham-specific
  signature can probabilistically identify the protocol. Pluggable
  transports raise the cost of this classification but do not
  eliminate it.

### 3.8 Sybil attacks on the relay pool

**Capability:** an attacker stands up a large number of relays to
increase the probability that all hops on a victim's path are
adversarial.

**Defence:**

- Directory authority vetting: relay submissions are reviewed
  before being added to the signed directory. This is a manual
  process today and is the rate limit on Sybil growth.
- Operator diversity in path selection: the path selector refuses
  two hops on the same path from the same declared operator.
- IP /16 diversity: the path selector refuses two consecutive hops
  in the same /16 IPv4 block.
- AS diversity: planned (B18 in CHECKLIST.md) — the selector will
  prefer paths spanning multiple autonomous systems.

**Residual surface:**

- An attacker willing to fund a long-running campaign across
  multiple jurisdictions and ASes can eventually accumulate hostile
  relays. Defence relies on the directory authority's vetting and
  the operator community's vigilance.

### 3.9 Compromise of the directory authority

**Capability:** the entity that signs the directory is coerced,
breached, or replaced.

**Defence:**

- The authority's signing key lives on a hardware token (YubiKey),
  signed offline.
- Backup is a Shamir 3-of-5 split stored in geographically and
  jurisdictionally distinct locations.
- Clients pin the authority's verifying key in the binary.
  Substituting a different authority requires a client update,
  which is itself signed.

**Residual surface:**

- A successful compromise of the signing key allows the attacker
  to publish a directory with hostile relays. This is detectable
  by clients comparing pinned key against published key. Client
  software updates are the recovery path, and pinning makes this
  one of the highest-impact failure modes — hence the operational
  caution.

### 3.10 Global Passive Adversary (out of scope)

**Capability:** an adversary with simultaneous visibility into a
sufficient majority of all network traffic globally — the canonical
"NSA-class" attacker.

**Position:** explicitly out of scope. **No low-latency anonymity
network in deployment today is provably safe against this
threat class.** Gotham makes the same disclaimer as Tor, Loopix,
Nym, and academic mix networks. The defence at this level is
out-of-band: high-latency mail-style systems, air-gapped
communication, or other techniques outside the network layer.

For users whose threat model includes a global passive adversary:
do not rely solely on Gotham. Combine with high-latency mailbox
protocols, in-person key exchange, and operational security
practices appropriate to that threat level.

### 3.11 Endpoint compromise (out of scope)

**Capability:** malware on the user's device, hardware keyloggers,
shoulder-surfing, coerced unlock.

**Position:** explicitly out of scope. Once the endpoint is
compromised, the network layer has no remaining property to
defend. Endpoint hygiene, full-disk encryption, secure boot,
hardware security keys, and operational discipline are the
defences at this layer.

### 3.12 Legal coercion (out of scope at protocol layer)

**Capability:** legal process compelling a user, a relay operator,
or the directory authority to disclose keys or modify behaviour.

**Position:** the protocol cannot defeat legal process. Operational
mitigations:

- For users: Crypto stores no information that would be useful
  under coercion beyond the master password (which protects the
  local SQLCipher database).
- For relay operators: see [GOTHAM-OPSEC.md](GOTHAM-OPSEC.md) §
  "Subpoena and legal process".
- For the directory authority: the signing key is on hardware and
  procedurally requires multiple parties (Shamir 3-of-5 backup
  plus offline signing).

---

## 4. Cryptographic primitives — assumptions

The threat model assumes the following primitives are secure under
their published security parameters:

| Primitive | Use | Status |
|-----------|-----|--------|
| X25519 | KEM half (classical) | Standard, well-studied |
| ML-KEM-768 | KEM half (post-quantum) | NIST FIPS 203 |
| HKDF-SHA256 | Key derivation | Standard |
| ChaCha20-Poly1305 | AEAD on payload + envelope | Standard, audited |
| Poly1305 | MAC chain on Sphinx header | Standard |
| Ed25519 | Directory signatures, audit log signatures | Standard |
| Noise XK | Link-layer mutual auth (relay-to-relay) | Standard (Noise framework) |

If any of these primitives is found to have a Critical vulnerability
that affects the relevant security goal, this entire threat model
must be re-evaluated.

---

## 5. Trust assumptions

| Trust | Required for |
|---|---|
| The user has a clean endpoint | G1, G2 (otherwise endpoint compromise applies, §3.11) |
| The user's first hop is honest enough to forward | G1 (otherwise availability degrades, but no metadata leak — only refusal) |
| At least one relay on a 3-hop path is honest | G1, G3 (Sphinx provides metadata protection if any single hop is honest) |
| The directory authority's signing key is not compromised | G1 indirectly (otherwise §3.9 applies) |
| The system clock on relays and clients is within the directory's validity window | Liveness, not security |

---

## 6. Known limitations and future work

- **GPA resistance:** see §3.10. Out of scope for v1.
- **Multi-sig directory:** v0.1 uses a single Ed25519 authority. v2
  is planned to use an N-of-M multi-sig scheme to reduce the impact
  of §3.9.
- **Sybil rate-limiting automation:** today manual via the directory
  authority. Automation under design.
- **Mobile cover traffic:** items 1.3.7 and 1.3.8 in
  [CHECKLIST.md](CHECKLIST.md) describe battery-aware degradation
  and backgrounded-app handling. These are required to extend the
  metadata-hygiene guarantees to mobile.
- **External audit:** the model in this document has been reviewed
  internally. Independent cryptographic audit (Trail of Bits /
  Quarkslab / Synacktiv / NCC Group) is the gate before the v0.2
  feature branch merges to `main`. Production deployment is
  contingent on completion of that audit.

---

## 7. Audit history

| Date | Reviewer | Scope | Outcome |
|------|----------|-------|---------|
| 2026-05-25 | Angel | Internal review against this document | — |
| TBD | External audit firm (TBD) | Cryptographic review of Gotham implementation | Pending |

This table will be appended on each subsequent review.

---

## 8. Reporting

If you believe you have identified a flaw in this threat model — a
threat class not covered, a defence claim that does not hold, or a
trust assumption that is not safely satisfied — please follow the
private reporting channel in [SECURITY.md](SECURITY.md). Threat
model bugs are taken as seriously as code bugs.

---

## References

- The protocol specification: [GOTHAM.md](GOTHAM.md).
- The inline `THREAT_MODEL` constant:
  [crypto-gotham/src/lib.rs](crypto-gotham/src/lib.rs).
- Academic background: Danezis & Goldberg, *Sphinx: A Compact and
  Provably Secure Mix Format*, IEEE S&P 2009. Piotrowska et al.,
  *The Loopix Anonymity System*, USENIX Security 2017. NIST FIPS
  203, *Module-Lattice-Based Key-Encapsulation Mechanism Standard*,
  2024.
