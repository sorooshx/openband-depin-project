# Gateway IP Privacy

**Status:** architectural target, not implemented.
**Priority:** Critical (C6 in [ROADMAP.md](ROADMAP.md)) — blocks beta to Iran.

> **Prerequisite:** the [blind-gateway TCP passthrough pivot](SECURITY.md#gateway-as-adversary-threat-model) (H4). That change ensures a gateway can't read destinations or content. The broker design below builds on top — it additionally hides *which gateway* the user is using and adds onion-routing for timing-analysis resistance.

## Why this document exists

[SECURITY.md](SECURITY.md) covers the single-exit-server case (protecting our own Hetzner IP). That problem is solvable by rotation + domain-fronted bootstrap. This document covers the **harder** problem: protecting the IPs of **volunteer gateway nodes** — the people running OpenBand on their Mac, router, or Starlink-connected device outside Iran who relay traffic for users inside Iran.

Once volunteer gateways exist, the failure mode is different:

- **Exit server compromise:** we rotate our server; users get new IPs from bootstrap.
- **Volunteer gateway compromise:** that person's IP is now on a blocklist forever, may be DDoSed, may face legal/social pressure if they're an Iranian expat, and the network loses throughput. If the gov enumerates many volunteer IPs, the whole relay pool dies.

Protecting volunteer IPs is the central unsolved problem of every censorship-resistance project (Tor, Snowflake, Psiphon, Lantern, Shadowsocks). There is no silver bullet. **The goal is to raise the cost of enumeration** so gov can't cheaply burn every gateway.

## Threat model addendum

Extends [SECURITY.md § Threat model](SECURITY.md#threat-model):

**New adversary capabilities to model:**
- Runs malicious user accounts to infiltrate the mesh from the inside.
- Runs malicious volunteer gateway nodes to observe which users connect.
- Subpoenas/coerces known volunteer operators in cooperating jurisdictions.
- DDoSes volunteer IPs once identified.

**New assets to protect:**
- Public IP of every volunteer gateway node.
- The mapping `user ↔ gateway` (who is using whom).
- The identity of volunteer operators.

**What the adversary is allowed to do in our model:**
- Download the app freely and reverse it.
- Subscribe as a real paying user.
- Run gateways and clients simultaneously.
- Passively observe network traffic at any ISP in Iran.

**What the adversary cannot do (assumptions):**
- Break TLS 1.3 with modern ciphers.
- Coerce CDN providers (Cloudflare, Fastly) at scale without consequences.
- Compromise the developer's signing keys.

---

## Layered defenses

Each layer is meaningful even if outer layers fail.

### L1. Clients never learn gateway IPs directly (rendezvous broker)
Client connects to a CDN-fronted broker. Broker authenticates the user, returns a signed WebRTC offer for *one* available gateway. Client + gateway establish direct P2P over WebRTC. Network observer sees only CDN traffic + encrypted P2P. Gateway IP is revealed only inside the ICE handshake, only to an authenticated client.

**This is how [Snowflake](https://snowflake.torproject.org/) (Tor) works.** It has survived sustained Iranian/Chinese blocking attempts since 2019.

### L2. Many small gateways, high churn
Target: hundreds of volunteer gateways, each serving 5–50 users. One burned = tiny blast radius. Residential IPs rotate naturally via DHCP; broker re-rolls the available pool every ~10 minutes.

### L3. Per-user gateway assignment
Each user is matched to a different subset of gateways. Seized phone leaks only that subset. To enumerate the full list, gov needs N subscribed accounts → enumeration cost scales with number of accounts purchased.

### L4. Authenticate before revealing any gateway info
Signed per-user subscription token (see [ROADMAP C3](ROADMAP.md)) gates the broker. No anonymous access. Gov agents can still subscribe, but bulk enumeration becomes expensive.

### L5. Invite-only social graph
User A introduces user B via out-of-band channel (Telegram DM, QR code). Only vetted users get tokens. Dramatically slows infiltration — a gov agent can't just sign up; they need an invite from a real user.

### L6. Onion-route across 2+ gateways
Client → Gateway₁ → Gateway₂ → exit. No single gateway knows `(user, destination)` simultaneously. Cost: 2× latency, half the throughput. This is Tor's core guarantee. Even if gov runs gateways, they can't deanonymize a flow without controlling both endpoints.

### L7. Traffic indistinguishability
A gateway's outbound traffic (to its exit) should be indistinguishable from a regular client's outbound traffic. Everyone speaks VLESS+Reality to the same-looking endpoints. Gov can't fingerprint "this IP is a gateway" from traffic patterns alone.

### L8. Gateway-side rate limiting + anomaly detection
A volunteer gateway that suddenly sees 1000 connections/minute from new clients is being probed. Rate-limit per-client-ASN; alert the operator via Telegram. Drop suspicious patterns.

---

## Concrete design: Snowflake-adapted rendezvous

**Components to build:**

```
┌─────────────────┐       ┌──────────────────┐       ┌─────────────────┐
│  Client (RN)    │       │  Broker          │       │  Volunteer       │
│  Iran           │       │  CDN-fronted     │       │  gateway         │
│                 │       │  (Cloudflare     │       │  (Mac / router)  │
│                 │       │   Worker)        │       │                  │
└────────┬────────┘       └────────┬─────────┘       └────────┬────────┘
         │                         │                          │
         │  1. auth + request GW   │                          │
         │  (fronted TLS)          │                          │
         ├────────────────────────>│                          │
         │                         │                          │
         │                         │  2. pick volunteer from  │
         │                         │     available pool       │
         │                         │<────── heartbeats ───────┤
         │                         │                          │
         │  3. signed WebRTC offer │                          │
         │<────────────────────────┤                          │
         │                         │                          │
         │  4. ICE exchange via broker (encrypted payload)    │
         │<───────────────────────>│<────────────────────────>│
         │                         │                          │
         │  5. direct WebRTC DataChannel (DTLS-encrypted)     │
         │<══════════════════════════════════════════════════>│
         │                                                    │
         │  6. gateway forwards to its own exit               │
         │         (Hetzner / own Starlink / VLESS / whatever)│
         │                                                    │
```

**Component responsibilities:**

### Broker (Cloudflare Worker or similar edge compute)
- HTTPS on CDN-fronted domain (no custom IP exposed)
- Accepts signed subscription tokens (see [ROADMAP C3](ROADMAP.md))
- Maintains volunteer registry (in-memory or KV store):
  - `nodeId`, last heartbeat, capacity, region, score
  - **Never stores mapping of `user → gateway`** (privacy by design)
- Matches one client to one gateway per session; returns signed WebRTC SDP offer
- Rate-limits per token (prevents one user enumerating the pool)

### Client (React Native, iOS + Android)
- Opens WebRTC PeerConnection
- Sends local SDP + ICE candidates to broker over fronted HTTPS
- Receives remote SDP + ICE for the matched gateway
- Establishes direct DTLS-encrypted DataChannel
- Proxies app traffic through DataChannel → gateway

### Volunteer gateway (Mac app, OpenWRT router, Linux daemon)
- Heartbeats to broker every 30s (stays in pool)
- Accepts WebRTC offers initiated by broker
- Forwards DataChannel traffic to its chosen exit (VLESS+Reality to Hetzner, or direct if operator has untraceable upstream)
- Reports bandwidth / session count back to operator

### Transport choice
- **WebRTC DataChannel** (DTLS over UDP, falls back to TCP).
  - Proven at scale (browser-native, used by Snowflake, Jitsi, WhatsApp).
  - NAT traversal via STUN/TURN for symmetric NATs.
- **STUN:** use Google's public STUN (`stun.l.google.com:19302`) as a default, plus self-hosted as backup.
- **TURN fallback:** self-hosted on Hetzner for clients behind symmetric NAT. Adds a proxy hop but keeps gateway IP still hidden from client.

### Key authentication
- Broker signs SDP offers with an Ed25519 key baked into the client.
- Client verifies signatures → broker can't be impersonated by network injection.
- Volunteers sign their heartbeats with a per-volunteer key issued at registration.

---

## Anti-patterns — things that look like solutions but aren't

### ❌ Obfuscating gateway IPs in the binary
Client needs to connect to *some* IP eventually. Whatever obfuscation the binary does, a reverser unwraps. The IP is in RAM at connection time. Obfuscation buys time, not security.

### ❌ "Most volunteers are legitimate, so mesh is safe"
Gov will run volunteer nodes themselves. Snowflake's academic paper covers this — the countermeasure is limiting what any single node can observe (onion routing, L6).

### ❌ Hoping IPs stay secret
Every paying user is potentially an informant. Assume gov has N subscribed accounts. Design so that N accounts leak ≤ N subsets of gateways (L3), not the whole pool.

### ❌ Trusting the broker to be trustworthy
The broker sees `(client, gateway)` pairs — that's the honey. Mitigations:
- Broker never *stores* pairs (stateless matching).
- Run multiple broker instances with different operators; client rotates.
- Audit broker code; make source-available.

### ❌ Relying on domain fronting alone
Google, Amazon, and Azure all killed direct domain fronting ~2018. Cloudflare and Fastly still work as of 2026 but may be next. Plan for layered fronting: the client supports N fronting providers, rotates on 403.

---

## Prior art

| Project | Mechanism | What OpenBand borrows |
|---------|-----------|------------------------|
| [Snowflake (Tor)](https://snowflake.torproject.org/) | Broker + WebRTC + volunteer browsers | L1 (broker + WebRTC), L2 (churn), L7 (traffic indistinguishability) |
| [meek (Tor)](https://gitlab.torproject.org/tpo/applications/tor-browser/-/wikis/Meek) | Domain fronting via CDNs | Fronting for broker access |
| [Psiphon](https://psiphon.ca/) | Multi-protocol, obfuscated first-contact | Multi-transport fallback |
| [Lantern](https://lantern.io/) | P2P volunteer "Giveaway" mode | Volunteer-operator model |
| [Shadowsocks](https://shadowsocks.org/) | Simple encrypted proxy | Baseline exit protocol |
| [I2P](https://geti2p.net/) | Fully P2P onion routing | L6 (onion routing) design reference |

## Open questions / research needed

- Can we run the broker as a Cloudflare Worker + Durable Object without losing latency? (Free tier limits apply.)
- What's the right N for "per-user gateway subset" size? Too small → poor UX when subset is offline. Too large → fast enumeration.
- WebRTC on RN: [`react-native-webrtc`](https://github.com/react-native-webrtc/react-native-webrtc) is maintained, but fits with our iOS `PacketTunnelProvider` sandbox restrictions?
- Legal exposure for volunteer gateway operators in the US/EU? Safe-harbor guarantees?
- Bandwidth compensation for high-usage volunteers — TON/USDT micropayments from user → gateway?

## Work breakdown (rough estimate)

Beyond what's already planned in [ROADMAP C1–C5](ROADMAP.md):

| Item | Est. | Depends on |
|------|------|------------|
| Broker MVP (Cloudflare Worker) | 1 week | C3 (token signing) |
| WebRTC integration on RN client | 1 week | — |
| Volunteer gateway WebRTC on Mac app | 4 days | — |
| Volunteer gateway on OpenWRT | 1 week | M1 |
| STUN/TURN self-hosted fallback | 2 days | — |
| Per-user subset assignment logic | 3 days | broker MVP |
| Onion routing (2-hop chain) — later phase | 2+ weeks | above all |

Realistic target: **v2.0 of OpenBand is when L1–L4 are shipped.** L5 and L6 are v3.0 work.

## What stays in the Hetzner server's role under this model

Under Snowflake-style rendezvous, the Hetzner server's role shrinks but doesn't disappear:

- **Exit of last resort** when no volunteer gateways are available (bootstrap phase).
- **TURN server** for NAT-symmetric clients.
- **Multi-broker coordinator** if we end up running several brokers.
- **Monitoring / abuse-report endpoint** for volunteer operators.

It is **not** a path that every user's traffic flows through — that's the whole point.
