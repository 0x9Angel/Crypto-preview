# Gotham — Relay Operator Security Guide (OPSEC)

This guide is for relay operators. It complements
[GOTHAM-DEPLOYMENT.md](GOTHAM-DEPLOYMENT.md) (the "how to install"
guide) with the operational discipline required to keep your relay
trustworthy.

The Gotham network's privacy properties hold only as long as the
relay pool is diverse, the relays are honest, and the relays are
not silently compromised. This document is the operator's
contribution to that guarantee.

**Last reviewed:** 2026-05-25

---

## Operator principles

Four principles guide every recommendation below.

1. **No surprise data retention.** A relay must hold no information
   that could correlate a sender with a recipient. Counters yes,
   per-packet records no.
2. **Minimum privilege everywhere.** The relay process, its files,
   and its network surface are reduced to the smallest set that
   still permits the relay to function.
3. **Defence in depth.** Even if one layer fails (a leaked log, a
   misconfigured firewall, a compromised SSH key), the next layer
   contains the damage.
4. **Transparency.** When you change something the community
   depends on, you say so. Silent operator changes erode trust faster
   than visible mistakes.

---

## Host hygiene

### Operating system

- Pick a stable Linux distribution with a long support window:
  Debian stable, Ubuntu LTS, RHEL/Rocky/Alma, or NixOS.
- Avoid rolling-release distributions (Arch, Tumbleweed) for
  production relays. The frequency of unexpected breakage is too
  high for unattended infrastructure.
- Enable unattended security updates. On Debian/Ubuntu:
  `unattended-upgrades` with `Unattended-Upgrade::Automatic-Reboot
  "true";` if your downtime tolerance allows.
- Keep the host firmware (BIOS / UEFI) up to date. VPS providers
  apply firmware updates to hypervisors; bare-metal hosts are your
  responsibility.

### Disk

- Full-disk encryption (LUKS) on bare metal; not always possible
  on cloud VMs, in which case rely on the provider's at-rest
  encryption and treat the disk as warm.
- Keep `/etc/gotham-relay/` permissions strict: directory `0700`,
  key files `0600`, owned by the `gotham-relay` service user.
- No swap, or encrypted swap. Default Linux swap can persist
  in-memory key material to disk.

### Time synchronisation

- Run a time daemon (`chrony` recommended, `systemd-timesyncd`
  acceptable, `ntpd` legacy but works).
- Drift target: < 100 ms relative to public stratum-2 servers.
- Why it matters: the directory authority emits time-bounded
  documents (`valid_after` / `valid_until`); a relay with a wildly
  skewed clock rejects the directory and stops accepting fresh
  paths.

### Logging discipline

The relay binary itself logs only counters (see GOTHAM-DEPLOYMENT.md
§ "Logging"). The host's other components also produce logs.
Operator responsibility:

- `journalctl` or `rsyslog` retention: 7-14 days is typical. Longer
  is acceptable but increases your obligation if subpoenaed.
- `iptables` / `nftables` connection-tracking: by default off; if
  you enable it for troubleshooting, disable again afterwards.
- Web server / proxy in front of the relay: not needed and not
  recommended. The relay speaks QUIC directly on UDP/443.
- SSH login records (`/var/log/auth.log`): keep. These records
  protect you (audit trail of admin access) and do not contain
  Gotham traffic.

---

## Network hygiene

### Firewall posture

Recommended inbound rules on the host firewall:

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 22 | TCP | Admin IPs only (or VPN) | SSH |
| 443 | UDP | Anywhere | Gotham QUIC |
| 443 | TCP | Anywhere | Gotham TLS fallback (when A6.2 ships) |

Everything else: drop, no ICMP reply (so you don't trivially
fingerprint as a Gotham relay vs a generic HTTPS service).

Recommended outbound rules:

| Destination | Protocol | Purpose |
|---|---|---|
| Public DNS or your own resolver | UDP/53, TCP/53 | Name resolution |
| Public NTP servers | UDP/123 | Time sync |
| Distribution mirrors | TCP/443 | Package updates |
| Other Gotham relays | UDP/443, TCP/443 | Forwarding |
| Directory authority | TCP/443 | Directory refresh |

Block outbound to internal RFC1918 ranges except for your own
management VPN, if any.

### Rate limiting and DoS

Gotham relays are deliberately stateless and small, but a
sufficiently determined attacker can flood your inbound. Defence:

- Enable your hosting provider's network-level DDoS protection if
  available (free tiers on Hetzner, OVH Anti-DDoS).
- Set a sane `max_entries` on the replay cache (default 1 000 000
  is good for most relays).
- Set kernel UDP buffer limits to non-default generous values
  (`sysctl net.core.rmem_max=26214400`).
- Monitor the `/health` endpoint counters. A sustained 10× spike in
  `packets_dropped_bad_mac` indicates an active flood.

### Tor / VPN considerations

A Gotham relay must NOT itself tunnel its outbound through Tor or
a VPN. The path selector assumes the public IP of each relay is
its true network identity for diversity calculations; lying about
this with a VPN weakens the diversity guarantee.

If you are required by your hosting provider's policy to tunnel
traffic through their VPN (rare), declare this clearly in your
operator metadata so the directory authority can decide how to
account for it.

---

## Key management

### Operational keys

Two keys live on the relay host (X25519 KEM + Ed25519 identity).
Both are required for the relay to function.

- **Never copy these keys off the host casually.** They are not
  intended to be portable.
- **For backup,** make a single encrypted snapshot stored offline
  (USB key in a safe). Restore is acceptable to recover from a
  total-host failure; treat the act of restoring as a key-rotation
  event and rotate within 30 days.
- **Do not commit keys to a version-control system.** Ever.

### Key rotation

- Routine rotation: annually for the X25519 KEM key.
- Triggered rotation: any of the following triggers an immediate
  rotation:
  - You suspect host compromise.
  - You are subpoenaed for key disclosure.
  - You restored from a backup.
  - You travelled with the host in transit (e.g. moved a bare-metal
    box between data centres).
- Rotation procedure: see GOTHAM-DEPLOYMENT.md § "Key rotation".

### Directory authority key (if you operate the authority)

If you are the directory authority operator (today: the project
team; long-term: a multi-party arrangement):

- Authority signing key MUST live on a hardware token (YubiKey 5
  or equivalent) with FIPS-mode firmware.
- Signing operations are manual, supervised, and logged in a
  separate audit log.
- Backup is a Shamir 3-of-5 secret split, stored in geographically
  distinct, jurisdictionally distinct locations.

---

## Compromise indicators

Watch for these signals on your relay host:

| Indicator | Likely cause | Action |
|---|---|---|
| Unexpected outbound connections to non-relay IPs | Malware exfiltration | Isolate, rotate keys, post-mortem |
| Unexpected processes running as `gotham-relay` | Process injection | Isolate, rotate keys |
| Disk usage growing without packet count growing | Hidden log somewhere | Find and remove |
| Suspicious SSH logins (`auth.log`) | Credential compromise | Rotate SSH keys, audit |
| Time drift > 30 minutes | NTP poisoning or clock attack | Investigate, force `chronyc makestep` |
| Sudden uptick in `packets_dropped_bad_mac` | Active probe or flood | Tighten rate limits |
| Configuration files modified outside your change window | Intrusion | Treat as compromise |

If you suspect compromise:

1. Disconnect the relay from the network (or stop the systemd unit).
2. Do NOT shut down the host immediately — keep memory live for
   forensics if you have the skill.
3. Notify the project's security contact via
   [SECURITY.md](SECURITY.md).
4. Rotate the relay keys before bringing the host back online.
5. Publish a brief post-mortem to the operators' mailing list.

---

## Subpoena and legal process

In most jurisdictions a Gotham relay operator has limited useful
information to disclose:

- No mapping of source to destination.
- No content (packets are encrypted end-to-end at multiple layers).
- No per-packet timestamps.
- Identities of peer relays, taken from the public directory.

What an operator CAN truthfully disclose:

- The fact that you operate a relay.
- The relay's public keys (these are already in the signed
  directory).
- The relay's IP address (already in the directory).
- Aggregate counters (total packets, by drop reason).

What you should do when served:

1. Consult a lawyer who understands telecommunications-intermediary
   law in your jurisdiction.
2. Document the request and the response.
3. If your jurisdiction allows a "transparency report" or the
   process is not under a gag order, publish that you received a
   request (even without details) so the community can see the
   pressure on the network.
4. Notify the project team via SECURITY.md if you believe the
   request reflects a systemic risk to the network.

---

## Operator metadata transparency

Every relay submits a public descriptor including:

- `operator` — free-form string declared by the operator
- `country` — ISO 3166-1 alpha-2
- `asn` — autonomous system number
- `uptime_pct` — rolling average

You SHOULD make the `operator` field meaningful (the name of your
NGO, university, foundation, or pseudonym) so users can assess
diversity. You SHOULD NOT lie about country or ASN; the path
selector relies on these for diversity, and lies reduce real
diversity.

If you operate multiple relays, declare them under the same
`operator` string so the path selector can refuse to use two of
them on the same path.

---

## Ethical considerations

Running a relay is a public service. Some ethical guidelines:

- **Do not log even if you could get away with it.** The temptation
  is real ("just in case"). Resist it. The network's value
  evaporates if operators silently log.
- **Do not censor.** Your relay forwards opaque ciphertext; you
  have no ability to censor by content. If you find yourself wanting
  to block certain destinations or peers, you should not be running
  a relay.
- **Do not use the relay for commercial purposes.** v1 has no
  payment model. Do not collect tips per relay, do not link the
  relay to a paid service. Mixing financial incentives with relay
  operation creates conflicts of interest.
- **Coordinate.** Join the operators' mailing list (when it exists)
  to get advance notice of network-level changes (directory
  authority rotation, protocol upgrades, scheduled maintenance).

---

## Further reading

- [GOTHAM.md](GOTHAM.md) — protocol specification.
- [GOTHAM-DEPLOYMENT.md](GOTHAM-DEPLOYMENT.md) — installation and
  operation.
- [GOTHAM-THREAT-MODEL.md](GOTHAM-THREAT-MODEL.md) — what the
  network resists and what it does not.
- [SECURITY.md](SECURITY.md) — reporting channel.

External references:

- [Tor Relay Operator Guide](https://community.torproject.org/relay/) —
  a different network, but the operator culture is closely related
  and many lessons transfer.
