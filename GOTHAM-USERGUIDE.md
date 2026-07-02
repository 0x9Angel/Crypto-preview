# Gotham — End-User Guide

This guide explains how to use Gotham as a Crypto user. It assumes
you have already installed Crypto on your desktop.

If you operate a Gotham relay (i.e. you run your own infrastructure
to participate in the network), read [GOTHAM-DEPLOYMENT.md](GOTHAM-DEPLOYMENT.md)
and [GOTHAM-OPSEC.md](GOTHAM-OPSEC.md) instead.

**Last reviewed:** 2026-07-03

---

## What is Gotham, in one paragraph

Gotham is the network layer that carries your encrypted messages.
End-to-end encryption (Signal protocol) protects the *content* of
what you write. Gotham is *designed* to protect the *metadata* — the
fact that you sent a message, when you sent it, and to whom. It is a
low-latency post-quantum mixnet, which means messages are wrapped in
fixed-size packets, routed through several volunteer-operated relays
with small random delays, and re-padded with cover traffic so an
observer cannot easily tell when you are active.

Gotham is now the **only** transport in Crypto. Earlier previews
could also route over Tor or Lokinet; those transports were removed
in v0.7 and Gotham carries all traffic.

> **Honest status (please read):** The *content* protection above
> (end-to-end encryption) is solid and works today. The *metadata*
> protection depends on a live network of relays spread across many
> different networks and hosting providers. At the time of writing
> that network is not yet live at scale — only a directory authority
> and a handful of relays are online, and they sit too close together
> on the network for the diversity rules to build a real route. So
> **real-world metadata anonymity is not yet delivered.** It becomes
> real once relays exist across many independent networks *and* an
> independent external audit has been completed. Treat Gotham's
> metadata protection as a design goal that is being brought online,
> not as a guarantee you can rely on today.

---

## The transport

There is nothing to choose here anymore. Crypto ships a single
transport — Gotham — and every message travels over it. Previous
previews let you pick between Gotham, Tor, and Lokinet; those extra
transports were removed in v0.7, so there is no transport selector in
current builds.

Gotham's link layer runs Noise XK over QUIC. It is marked "beta"
until an independent external audit completes.

> **Note:** The transport is not what keeps your messages private
> *in content* — that is the end-to-end encryption layer, which
> applies to every message. Gotham's job is to hide *who can see
> that a message was sent* (its metadata), which, as noted above,
> is a design goal that is still being brought online.

---

## What Gotham hides, and what it does not hide

### What Gotham is designed to hide

*(These properties hold once a live network of relays spanning many
independent networks exists. Today that network is not yet live at
scale, so the items below are the design intent, not a delivered,
real-world-proven guarantee.)*

- The fact that you sent a message at a particular instant.
- Who you sent it to (the recipient is meant to be opaque to the
  network).
- Your usage pattern (frequency, time-of-day, conversation graph).
- The size of your messages (all packets are 2048 bytes).

### What Gotham does NOT hide

- That you are using a Gotham-capable application (the first hop
  knows you are sending mixnet packets, just not which packets
  contain real traffic).
- The content of your messages from your endpoint (if your device
  is compromised, attackers see the plaintext).
- Your offline activity (Gotham only protects packets in flight).
- Your identity from the recipient (the recipient knows it is you,
  by design — that is the whole point of you wanting to message them).

### Threats Gotham resists

- Passive network observers (your ISP, café Wi-Fi, network taps).
- Single-relay compromise at any tier.
- Replay attacks (5-minute MAC cache per relay).
- Tagging attacks (each hop's payload is protected by a
  non-malleable wide-block cipher, so tampering with a packet
  corrupts it rather than marking it).
- Quantum adversaries harvesting traffic today to decrypt later
  (post-quantum hybrid key exchange: X25519 combined with
  ML-KEM-768 at each hop).

### Threats Gotham does NOT resist

- Global passive adversaries with visibility into a majority of the
  internet (the canonical "NSA-class adversary"). For this threat
  level, no low-latency anonymity network in existence today is
  provably safe.
- Majority compromise of the relay pool (more than ~60% hostile
  relays).
- Endpoint compromise (malware on your device, shoulder-surfing,
  keyloggers).
- Legal coercion of you personally to reveal your keys.

---

## Setting up Gotham for the first time

When you first enable Gotham in **Settings → Profile → Transport**,
the application performs a one-time bootstrap:

1. Generates a long-term X25519 identity key for your account.
   This key is stored in the SQLCipher-encrypted local database
   and never leaves your device.
2. Downloads the signed Gotham directory document. This lists the
   currently-active relays and is signed by the directory authority.
3. Validates the signature against the pinned authority public key.
4. Attempts to select three relays for path construction, enforcing
   the diversity rules (the three relays must sit on distinct
   networks and be run by distinct operators, with the entry relay
   different from the exit).

You will see a progress bar during this bootstrap. Total time is
typically 2-5 seconds on a desktop, longer on a slow network.

> **Note:** Because the live relay set is currently too small and
> too clustered on the network, path construction may not yet
> succeed on the public network — the diversity rules will correctly
> refuse to build a route rather than build an unsafe one. This is
> expected while the network is still being brought online.

---

## Sharing your Gotham identity

To receive Gotham messages from another Crypto user, that user
needs your **Gotham public key** (32 bytes, displayed as 64
hexadecimal characters).

To share it:

1. Go to **Settings → Profile → Identity**.
2. Click "Copy Gotham public key" or scan the QR code on another
   device.
3. Send the hex string or QR to your contact via any secure side
   channel (an existing Signal session, an in-person meeting, a
   signed PGP email, etc.).

Your contact then adds your key in **Contacts → Add → Paste Gotham
public key**.

> **Important:** sharing your Gotham public key via the network you
> are trying to anonymise (sending it over an unrelated app on the
> same device) is fine. Sharing it over a public, untrusted channel
> (a tweet, a forum post) makes your identity correlatable across
> conversations — which is rarely what you want.

---

## Sending a message

Once both sides have exchanged Gotham keys, sending a message is
just like any chat app: open the conversation, type, hit send.

Under the hood:

1. The application encrypts the message with the Double Ratchet
   session you share with the recipient.
2. The encrypted ciphertext is wrapped in a sealed-sender envelope
   that hides your identity from the network.
3. The envelope is placed in a 2048-byte Gotham packet, routed
   through three relays with small Poisson-distributed delays at
   each hop.
4. The final relay delivers locally to the recipient's embedded
   relay.
5. The recipient's application unseals, decrypts, and renders the
   message.

On a healthy network the target median round-trip is 100-300 ms
(for comparison, Tor typically runs 800-2000 ms). This is achieved
by tuning the cover traffic and Poisson delay parameters for
chat-sized payloads, not file transfer. These are design targets;
end-to-end latency on the public network cannot be measured until a
live, diverse relay set is online.

---

## Cover traffic

Your client emits a small stream of indistinguishable "cover"
packets at random intervals, even when you are not actively sending
real messages. Real messages are slotted into this same stream, so
the aim is that a network observer cannot tell when you are typing
and when you are idle. (This property depends on a live, diverse
relay network to be meaningful in the real world — see the honest
status note near the top of this guide.)

Default cover-traffic mode is **balanced**: roughly one cover packet
per 30 seconds on average. You can adjust this in **Settings →
Privacy → Cover traffic level**:

| Mode | Approximate rate | Best for |
|---|---|---|
| **low-latency** | 1 packet / 10 s | Foreground use, charging, fast network |
| **balanced** (default) | 1 packet / 30 s | Most users |
| **paranoid** | 1 packet / 5 s | High-threat environments, willing to accept higher bandwidth |

Cover traffic uses approximately 5-50 KB per hour depending on
mode. On metered connections this is generally invisible. (Crypto
currently ships desktop clients only — Linux, macOS, and Windows;
there are no mobile clients yet.)

---

## File attachments

File attachments use the same Gotham transport as messages, but the
file is split into multiple 2048-byte packets and reassembled on
the recipient side.

Large files (over a few megabytes) are slow over Gotham — this is
intentional. The mixnet is tuned for interactive chat, not bulk
transfer, and all attachments travel over Gotham like everything
else.

---

## Troubleshooting

### "Bootstrap failed"

Most common cause: the directory authority is unreachable from your
current network. Verify:

- Your internet connection is up.
- UDP outbound (QUIC) is not blocked by a corporate firewall.
  Gotham's link layer is Noise XK over QUIC, which needs outbound
  UDP; some networks block it.
- The system clock is reasonably accurate. The directory has
  `valid_after` / `valid_until` timestamps and rejects out-of-range
  clocks.

### "All three hops failed"

A path was selected but at least one relay rejected the connection.
The application automatically retries with a fresh path. Because
Gotham is the only transport, there is no other transport to fall
back to; if retries keep failing, the message stays queued and is
retried later.

### "No usable path" / "Not enough diverse relays"

The directory does not currently list enough relays on distinct
enough networks to satisfy the diversity rules, so no safe path can
be built. This is expected while the public relay network is still
being brought online (see the honest status note near the top of
this guide) and is not a bug in your client — the client is
correctly refusing to build an unsafe route.

### Messages are sometimes delayed

Expected behaviour. Gotham deliberately adds Poisson-sampled delays
at each hop to defeat timing correlation. Median is 100-300 ms but
individual messages can take longer. This is the cost of metadata
privacy.

### "Replay detected" warning

The same packet was seen twice by a relay within its 5-minute MAC
cache window. Almost always a transient network glitch (a duplicate
retransmission). If you see this repeatedly for the same recipient,
report a bug — it may indicate a session-state desynchronisation.

---

## Privacy hygiene checklist

To get the most out of Gotham:

- [ ] Keep the application up to date — security patches are
      released on the SLA documented in [SECURITY.md](SECURITY.md).
- [ ] Use a strong, unique master password (Argon2id-protected,
      12+ characters, mixed character classes).
- [ ] Pair Gotham with full-disk encryption on your device.
- [ ] Verify each contact's identity out-of-band using the 60-digit
      **safety number** shown in the conversation (compare it in
      person or over another trusted channel). The app forces this
      step for new contacts, marks a contact "verified" once you
      confirm, and alerts you if a contact's key later changes —
      which can indicate a machine-in-the-middle attempt.
- [ ] Periodically check the directory authority pubkey fingerprint
      against the canonical value published at
      `https://github.com/0x9Angel/Crypto/blob/main/GOTHAM.md`.

---

## Where to go next

- For protocol details: [GOTHAM.md](GOTHAM.md).
- For the threat model: [GOTHAM-THREAT-MODEL.md](GOTHAM-THREAT-MODEL.md).
- To run your own relay: [GOTHAM-DEPLOYMENT.md](GOTHAM-DEPLOYMENT.md).
- For security policy and reporting: [SECURITY.md](SECURITY.md).
- For everything: [GOTHAM-DOSSIER.md](GOTHAM-DOSSIER.md).
