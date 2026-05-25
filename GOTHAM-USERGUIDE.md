# Gotham — End-User Guide

This guide explains how to use Gotham as a Crypto user. It assumes
you have already installed Crypto on your desktop.

If you operate a Gotham relay (i.e. you run your own infrastructure
to participate in the network), read [GOTHAM-DEPLOYMENT.md](GOTHAM-DEPLOYMENT.md)
and [GOTHAM-OPSEC.md](GOTHAM-OPSEC.md) instead.

**Last reviewed:** 2026-05-25

---

## What is Gotham, in one paragraph

Gotham is the network layer that carries your encrypted messages.
End-to-end encryption (Signal protocol) protects the *content* of
what you write. Gotham protects the *metadata* — the fact that you
sent a message, when you sent it, and to whom. It is a low-latency
post-quantum mixnet, which means messages are wrapped in fixed-size
packets, routed through several volunteer-operated relays with
small random delays, and re-padded with cover traffic so an observer
cannot tell when you are active.

Crypto can also send messages over Tor v3 or Lokinet. Gotham is the
recommended default starting with v1.2.x but remains in beta until
the external audit completes (target: 2027). For production-critical
communication today, prefer Tor v3.

---

## Choosing a transport

In the application, go to **Settings → Profile → Transport**. You
will see three radio options:

| Option | When to choose it |
|---|---|
| **Gotham** (default) | Best for general use. Low latency, post-quantum, metadata-hiding. Marked "beta" until the external audit completes. |
| **Tor v3** | Choose if your threat model assumes a near-global adversary or you need the most-audited anonymity transport available. Higher latency. |
| **Lokinet** | Niche option for environments where Lokinet is already deployed. |

You can switch transport at any time. Existing conversations continue
to work; new messages use the newly-selected transport.

> **Note:** Switching transport does not change *who* can read your
> messages — that is the end-to-end encryption layer, which is the
> same regardless of transport. Switching transport changes only
> *who can see that a message was sent*.

---

## What Gotham hides, and what it does not hide

### What Gotham hides

- The fact that you sent a message at a particular instant.
- Who you sent it to (the recipient is opaque to the network).
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
- Tagging attacks (Sphinx MAC chain).
- Quantum adversaries harvesting traffic today to decrypt later
  (post-quantum hybrid key exchange).

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
4. Selects three initial relays for path construction.

You will see a progress bar during this bootstrap. Total time is
typically 2-5 seconds on a desktop, longer on a slow network.

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
identical to sending via Tor or any other transport: open the
conversation, type, hit send.

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

Median round-trip on a healthy network is 100-300 ms (compared to
800-2000 ms for Tor v3). This is achieved by tuning the cover
traffic and Poisson delay parameters for chat-sized payloads, not
file transfer.

---

## Cover traffic

When Gotham is enabled, your client emits a small stream of
indistinguishable "cover" packets at random intervals, even when you
are not actively sending real messages. This is what makes it
impossible for a network observer to tell when you are typing and
when you are idle.

Default cover-traffic mode is **balanced**: roughly one cover packet
per 30 seconds on average. You can adjust this in **Settings →
Privacy → Cover traffic level**:

| Mode | Approximate rate | Best for |
|---|---|---|
| **low-latency** | 1 packet / 10 s | Foreground use, charging, fast network |
| **balanced** (default) | 1 packet / 30 s | Most users |
| **paranoid** | 1 packet / 5 s | High-threat environments, willing to accept higher bandwidth |

Cover traffic uses approximately 5-50 KB per hour depending on
mode. On metered connections this is generally invisible. On
battery-constrained mobile devices, the application automatically
degrades to a quieter rate (see [CHECKLIST.md](CHECKLIST.md) item
1.3.7).

---

## File attachments

File attachments use the same Gotham transport as messages, but the
file is split into multiple 2048-byte packets and reassembled on
the recipient side.

Large files (over a few megabytes) are slow over Gotham — this is
intentional. The mixnet is tuned for interactive chat, not bulk
transfer. For large attachments, the application transparently
falls back to a Tor Hidden Service direct connection, with the
fallback indicated by a small icon next to the file in the chat.

---

## Troubleshooting

### "Bootstrap failed"

Most common cause: the directory authority is unreachable from your
current network. Verify:

- Your internet connection is up.
- UDP/443 outbound is not blocked by a corporate firewall (try
  switching to **Tor v3** transport as a workaround while you
  investigate).
- The system clock is reasonably accurate. The directory has
  `valid_after` / `valid_until` timestamps and rejects out-of-range
  clocks.

### "All three hops failed"

A path was selected but at least one relay rejected the connection.
The application automatically retries with a fresh path; if three
retries in a row fail, the application falls back to the last
working transport.

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
- [ ] Verify Gotham public keys out-of-band with new contacts
      before relying on the channel.
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
