# Gotham — Relay Deployment Guide

This document is for operators who want to run a Gotham relay as
part of the public mixnet, or as part of a private deployment for
their organisation.

If you are a regular Crypto end-user, you do not need to read this
document — the application embeds a small client-side relay
automatically. See [GOTHAM-USERGUIDE.md](GOTHAM-USERGUIDE.md).

**Status:** Pre-GA. The public Gotham network is not yet deployed.
This document describes the intended deployment model for the
production network, targeted late 2027.

**Last reviewed:** 2026-05-25

---

## Why operate a relay

Relays are the backbone of the mixnet. The privacy properties of
Gotham (metadata-hiding, traffic-analysis resistance) hold only as
long as the relay pool is diverse, independent, and large enough
that no single party controls a majority.

You should consider running a relay if you are:

- A volunteer in the privacy / cypherpunk community.
- A university, NGO, or research lab interested in deploying
  infrastructure that supports civil liberties.
- An organisation (defence integrator, ministry, critical-infra
  operator) deploying a private Gotham instance for internal use.

You should NOT run a relay if:

- You cannot commit to running it for at least 6 months.
- You cannot keep the host's security baseline current (timely
  patches, OS updates).
- You expect the relay to be profitable. There is no payment model
  in v1.

---

## Hardware and bandwidth requirements

A Gotham relay is intentionally lightweight. It runs comfortably on
a small VPS.

### Minimum (entry or exit tier)

- 1 vCPU
- 1 GB RAM
- 20 GB SSD
- 100 Mbit/s symmetric, unmetered or generous monthly quota

### Recommended (mix tier)

- 2 vCPU
- 2 GB RAM
- 40 GB SSD
- 1 Gbit/s symmetric

### Network

- A public IPv4 address (IPv6-only is not yet supported in v0.1).
- Open inbound UDP/443 and TCP/443 for the relay protocol.
- Open outbound to peers in the directory.
- A reverse DNS record matching your hostname is recommended for
  reachability diagnostics.

### Hosting providers

Diversity matters. The path selector rejects two consecutive hops
from the same /16 IPv4 block or the same declared operator. For the
public network we aim for relays across at least:

- 3 distinct legal jurisdictions
- 3 distinct hosting providers
- 3 distinct autonomous systems (ASes)

Recommended providers (no commercial relationship — these are
suggestions based on community experience):

- **Hetzner** (DE) — cheap, generous bandwidth, AS24940.
- **OVH / OVHcloud** (FR) — sovereign EU, AS16276.
- **Vultr** (multiple) — wide geographic coverage.
- **Linode / Akamai** (multiple) — established, stable.
- **DigitalOcean** (multiple) — easy onboarding.

Avoid hosting at major cloud providers (AWS, Google Cloud, Azure)
unless you have a specific reason. They concentrate traffic at a
few autonomous systems and undermine path diversity.

---

## Installation

### 1. Provision the host

Provision a Linux VPS (recommended: Debian 12, Ubuntu 24.04 LTS, or
Fedora Server 41). Follow your provider's hardening basics:

- SSH key authentication only, password login disabled.
- Unattended security updates enabled (`unattended-upgrades` or
  equivalent).
- A non-root user with sudo for administration.
- A minimal firewall: only SSH (from your admin IPs), UDP/443, and
  TCP/443 inbound.

### 2. Build the relay binary

You have two options:

#### Option A — build from source (recommended for trust)

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# Clone the source
git clone https://github.com/0x9Angel/Crypto.git
cd Crypto

# Build the relay
cargo build --release -p crypto-gotham-relay

# The binary is at target/release/gotham-relay
```

This takes 10-20 minutes depending on hardware.

#### Option B — install a signed pre-built binary

When public release artefacts are available (target: Q1 2027):

```bash
# Verify the signature first
cosign verify-blob \
  --certificate-identity-regexp "https://github.com/0x9Angel/Crypto" \
  --signature gotham-relay-1.x.y.sig \
  gotham-relay-1.x.y

# Install
sudo mv gotham-relay-1.x.y /usr/local/bin/gotham-relay
sudo chmod +x /usr/local/bin/gotham-relay
```

Reproducible-build verification is documented separately in
[docs/REPRODUCIBLE-BUILDS.md](docs/REPRODUCIBLE-BUILDS.md).

### 3. Generate the relay identity

A Gotham relay has two long-term keys:

- An **X25519 KEM key** used by clients to encapsulate to this relay
  in the Sphinx header.
- An **Ed25519 identity key** used for signing the relay's own
  attestations.

Generate both with the bundled tool:

```bash
sudo mkdir -p /etc/gotham-relay
sudo gotham-relay keygen \
  --x25519-out /etc/gotham-relay/x25519.key \
  --ed25519-out /etc/gotham-relay/ed25519.key
sudo chmod 600 /etc/gotham-relay/*.key
sudo chown gotham-relay:gotham-relay /etc/gotham-relay/*.key
```

Print the corresponding public keys — you will need them to submit
your relay to the directory:

```bash
gotham-relay pubkeys --x25519 /etc/gotham-relay/x25519.key
gotham-relay pubkeys --ed25519 /etc/gotham-relay/ed25519.key
```

### 4. Configure the relay

Create `/etc/gotham-relay/config.toml`:

```toml
# /etc/gotham-relay/config.toml

[identity]
x25519_key_path = "/etc/gotham-relay/x25519.key"
ed25519_key_path = "/etc/gotham-relay/ed25519.key"

[network]
listen_addr = "0.0.0.0:443"        # both UDP and TCP
public_addr = "203.0.113.42:443"   # your public IP

[relay]
tier = "mix"                       # "entry" | "mix" | "exit"
country = "FR"                     # ISO 3166-1 alpha-2
operator = "ngo-yourname"          # public string, free-form
asn = 16276                        # your hosting provider's ASN

[replay_cache]
ttl_seconds = 300                  # 5 minutes
max_entries = 1000000              # bound the cache

[delay]
mean_micros = 50000                # 50 ms average Poisson delay

[logging]
level = "info"                     # never log per-packet data
```

The configuration is intentionally minimal. The relay does not need
to know about other relays — it only processes incoming packets.

### 5. Install the systemd service

A reference unit file is provided in `crypto-gotham-relay/deploy/`.
Copy it to your system and enable it:

```bash
sudo cp crypto-gotham-relay/deploy/gotham-relay.service \
        /etc/systemd/system/gotham-relay.service
sudo systemctl daemon-reload
sudo systemctl enable --now gotham-relay
sudo systemctl status gotham-relay
```

The service runs as a dedicated `gotham-relay` user with no shell
and a restricted filesystem view (see the unit file for details).

### 6. Verify reachability

From an external host:

```bash
# Check UDP/443 reachability (QUIC handshake)
nmap -sU -p 443 your-relay.example.com

# Check TCP/443 fallback
nmap -p 443 your-relay.example.com
```

Both should report `open`. If they report `filtered`, your hosting
provider's network ACL is blocking the port; contact support.

### 7. Submit the relay to the directory

Once the production directory authority is live (target: Q1 2027):

```bash
gotham-relay submit \
  --config /etc/gotham-relay/config.toml \
  --directory-url https://directory.gotham.<production-domain>/submit
```

The submission contains the public keys, the declared tier, the
declared country and operator, and a self-signed attestation. The
directory authority reviews it (currently manual, with plans for a
vetted automated workflow), and on approval the relay appears in
the next signed directory snapshot.

---

## Operating the relay

### Monitoring

The relay exposes a `/health` endpoint over HTTP on a configurable
admin port (default: localhost-only, port 9999). The endpoint
returns counters only — never per-packet data, never source IPs,
never destination IPs.

Example response:

```json
{
  "uptime_seconds": 86400,
  "packets_forwarded_total": 123456,
  "packets_dropped_replay": 12,
  "packets_dropped_bad_mac": 3,
  "packets_dropped_malformed": 0,
  "replay_cache_len": 4567,
  "version": "1.2.1"
}
```

Hook this into Prometheus, Datadog, or any other monitoring system.
See `crypto-gotham-relay/deploy/prometheus-scrape-example.yml` for
a sample scrape configuration.

### Logging

The relay logs at the `info` level by default. Log lines contain
only:

- Timestamps
- Counters (packets in / out / dropped, replay cache size)
- Lifecycle events (startup, shutdown, key rotation)

The relay NEVER logs:

- Source or destination IP addresses
- Packet contents or sizes
- Hashes of packet contents (these could correlate flows)
- Per-packet timing

If you find a log line containing any of the forbidden items, treat
it as a security bug and report it via [SECURITY.md](SECURITY.md).

### Updates

When a new version is released:

1. Review the [CHANGELOG.md](CHANGELOG.md) entry. Pay special
   attention to anything under **Security**.
2. Verify the new binary signature.
3. Stop the running relay (`sudo systemctl stop gotham-relay`).
4. Replace the binary.
5. Restart (`sudo systemctl start gotham-relay`).

Total downtime: a few seconds. Clients automatically retry through
the cache of last-known-good directory entries during this window.

### Key rotation

X25519 KEM keys are rotated annually (or sooner if compromise is
suspected). To rotate:

```bash
# Generate a new key
sudo gotham-relay keygen \
  --x25519-out /etc/gotham-relay/x25519-new.key

# Trigger a graceful in-place rotation
sudo gotham-relay rotate \
  --new-x25519 /etc/gotham-relay/x25519-new.key \
  --config /etc/gotham-relay/config.toml
```

The relay continues to accept the old key for a grace period (default
1 hour) while clients refresh their directory.

---

## Decommissioning

If you need to stop running the relay:

1. Announce your intent via the operators' mailing list (or, for
   private deployments, internally) at least 7 days in advance.
2. Submit a "going offline" attestation to the directory authority.
3. After the announced date, stop the systemd service and securely
   wipe the key files:

```bash
sudo systemctl stop gotham-relay
sudo systemctl disable gotham-relay
sudo shred -u /etc/gotham-relay/*.key
sudo rm -rf /etc/gotham-relay /var/lib/gotham-relay
```

The directory authority removes the descriptor in the next signed
snapshot.

---

## Legal considerations

Running a Gotham relay carries the same legal status as running a
Tor middle relay in most jurisdictions: you do not see plaintext
traffic, you do not know endpoints, and you forward fixed-size
ciphertext between other relays.

Specific considerations:

- **France:** Relay operators may receive judicial requests for
  logs. The relay produces no logs that link source and destination,
  so there is nothing to disclose beyond the operator's own
  identity and the fact of operating the relay.
- **Germany:** Telemediengesetz § 8 (telecoms intermediary
  privilege) generally applies.
- **United States:** Section 230 protections generally apply.
- **United Kingdom, Australia, Canada:** consult local counsel
  regarding metadata-retention obligations.

For private deployments inside a single organisation, the legal
analysis is different — the operator's responsibilities depend on
the data being carried. Consult counsel.

---

## Further reading

- [GOTHAM.md](GOTHAM.md) — protocol specification.
- [GOTHAM-OPSEC.md](GOTHAM-OPSEC.md) — operational security best
  practices specific to relay operators.
- [GOTHAM-THREAT-MODEL.md](GOTHAM-THREAT-MODEL.md) — full threat
  model including operator considerations.
- [SECURITY.md](SECURITY.md) — security advisories and reporting
  channels.
