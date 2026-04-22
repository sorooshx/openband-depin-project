# OpenBand — A DePIN Mesh Network with Satellite Uplinks for Censorship Circumvention

**Version 0.1 · Pre-beta Whitepaper**
**Date: April 2026**
**Website:** [https://www.openband.io](https://www.openband.io)

---

## Abstract

OpenBand is a censorship-circumvention system targeting users in jurisdictions with state-level traffic filtering. The architecture rests on three mutually reinforcing pillars:

1. **DePIN (Decentralized Physical Infrastructure Network)** — gateways are a distributed pool of volunteer-operated devices, primarily **OpenWRT routers** (with Starlink, fiber, or LTE upstream), supplemented by Mac laptops in development environments. No single operator, company, or jurisdiction controls the network.
2. **Local-area mesh networking** — phones inside a censored region discover nearby gateways over UDP broadcast, elect one automatically by score, and tunnel traffic through it. Zero user configuration, no pre-shared knowledge of gateway IPs.
3. **Satellite and alternative uplinks** — gateway nodes can route upstream through any non-terrestrial path (Starlink, Iridium, and other LEO/MEO services) in addition to fiber, cable, and cellular. A state adversary controls its border infrastructure but cannot block satellite orbits.

Together, these defeat the single failure mode of centralized VPNs: no one IP, ASN, operator, ISP, or jurisdiction can take down the network. End-to-end encryption (VLESS+Reality) between the user's phone and an exit server abroad provides both traffic-content privacy and DPI resistance via TLS masquerading.

This document describes the threat model, architecture, mesh protocol, gateway trust model, DePIN operator model, satellite-backhaul role, and the current security posture. It is deliberately honest about what is not yet implemented — OpenBand is pre-beta, and several critical properties (multi-exit rotation, per-user credentials, encrypted mesh layer) are planned but not shipped. Readers in the security and anti-censorship communities are invited to audit, contribute, and critique.

---

## 1. Introduction

State-level internet filtering has intensified globally since 2022. Authorities in censorship-heavy jurisdictions block most Western platforms, throttle TLS traffic to unknown destinations, fingerprint VPN protocols, and pressure ISPs to identify users of popular circumvention tools. Commercial VPNs centralize risk on a small number of exit IPs that are enumerated and blocked within days of launch. Tor's bridges are regularly discovered via active probing. Most existing tools optimize for either throughput *or* resistance, not both.

OpenBand takes a different approach. Instead of hiding a small number of high-value servers, we distribute the exit surface across **a DePIN-style network of volunteer-operated gateway nodes** (OpenWRT routers and Starlink-connected terminals as the primary deployment, Mac laptops for development, running in uncensored — or *less*-censored — jurisdictions), reached by phones inside the target region through a **local WiFi mesh**. Crucially, those gateway nodes can backhaul via **satellite** (Starlink, Iridium) in addition to traditional ISPs, removing the national-border chokepoint that terrestrial circumvention tools depend on. The adversary's task shifts from "block one IP" to "enumerate and block thousands of residential IPs plus block LEO satellite constellations" — expensive, constantly moving, and in the satellite case, geopolitically impossible. This is the strategy that has kept Tor's Snowflake pluggable transport viable for years, extended with satellite backhaul and DePIN incentive alignment.

OpenBand is implemented as:

- A **React Native mobile app** (Android 10+, iOS 11+) — **client only** on iOS by design; Android can additionally act as an ephemeral hotspot egress in blackout-mode (Phase 3, optional)
- A **SwiftUI macOS app** that acts as a gateway node for development and small-scale testing
- An **OpenWRT router package** for dedicated always-on gateways — the primary production deployment target (in active development, see Roadmap § H4c / M1)
- **Exit servers** running VLESS+Reality on standard ports

The codebase is **proprietary** (all rights reserved) during pre-launch. Documentation, whitepaper, and protocol specifications are public under **CC-BY-4.0**. The security claims in this document are bounded by what is currently implemented — we explicitly separate shipped guarantees from planned ones. The threat model is published so users (and auditors) are not asked to trust binary code blindly: every claim is documented and testable against the public specs.

---

## 2. Threat Model

### 2.1 Adversary capabilities (assumed)

A state-level adversary with control of the national internet infrastructure. Specifically:

- **DPI on all transit ISPs.** Can inspect every packet crossing the national border.
- **IP blocklists.** Can add any IP, ASN, or SNI to a national block within minutes.
- **Active probing.** Can send crafted packets to any exposed endpoint to identify circumvention infrastructure.
- **Binary reverse engineering.** Can extract any secret compiled into the mobile app binary.
- **Legal coercion.** Can compel domestic ISPs to identify subscribers. Cannot reliably compel foreign CDN operators (Cloudflare, Fastly, Amazon).
- **Infiltration.** Can purchase subscriptions as a user and run its own gateway nodes.
- **Local WiFi control.** Can operate public WiFi networks (airports, cafés) and observe / inject on those networks.

### 2.2 Assets we protect

1. **User's ability to reach the open internet** (the headline property).
2. **User identity** — which user is using OpenBand at a given time.
3. **User's browsing history** — which sites a user visits through OpenBand.
4. **Gateway IPs** — volunteer operators' home IPs must not become public blocklist fodder.
5. **Server credentials** — so one leaked user account doesn't compromise all users.
6. **The mesh graph** — who is connected to whom, spatial proximity patterns.

### 2.3 Assets we explicitly do not protect (at this stage)

- **Full metadata resistance** (timing + volume analysis). Addressed only partially by onion routing (roadmap).
- **Identity of volunteer operators** against determined investigation.
- **Content of what users actually do online** beyond what the exit protocol (VLESS+Reality) already protects.

### 2.4 The adversarial gateway

A critical assumption: **gateway operators may be hostile**. An adversary can purchase a subscription, run a gateway, and log everything that passes through it. The security model must assume this and not rely on gateway honesty for anything beyond liveness. This is the Tor security model, adapted.

---

## 3. Background and Prior Art

OpenBand borrows from several lines of research and engineering:

| Work | What we borrow |
|------|----------------|
| **Tor** | Onion-routing mental model; adversarial relay model; bridge distribution |
| **Snowflake (Tor)** | Volunteer proxy model, domain-fronted broker, WebRTC ephemeral proxies |
| **VLESS + Reality** (Xray / sing-box) | Exit protocol — TLS-in-TLS masquerading as a real destination SNI; resists active probing |
| **meek / Psiphon** | CDN-fronted control channel |
| **Shadowsocks** | Lightweight encrypted tunnel baseline |
| **WireGuard** | Phone↔gateway LAN-hop encryption with nested Reality end-to-end (H4c, in progress); inter-router fabric encryption (M2) |
| **Briar / Secure Messenger projects** | Mesh networking and discovery over local radio |

The novel contributions of OpenBand are:

1. **Mobile-first mesh.** Smartphones elect a local gateway automatically via UDP multicast discovery; users do not configure anything.
2. **End-to-end exit crypto even across volunteer hops.** The Reality handshake is (or will be) between the user's phone and the exit server — volunteer gateways act as byte-pipes when possible.
3. **Zero-touch onboarding** via OS-native WiFi suggestion APIs (WifiNetworkSuggestion / NEHotspotConfigurationManager), matching production deployment scenarios where users have no direct internet.

---

## 4. System Overview

### 4.1 Topology

```
┌────────────────────────────────────────────────────────────────┐
│  LAN — volunteer-operated access point or shared WiFi          │
│                                                                │
│  ┌──────────┐    UDP :5555 mesh     ┌──────────┐               │
│  │  iPhone  │ ←───JSON discovery──→ │  Android │               │
│  │  client  │                       │  client  │               │
│  └────┬─────┘                       └────┬─────┘               │
│       │                                  │                     │
│       └──────────┬───────────────────────┘                     │
│                  │  encrypted tunnel to elected gateway        │
│                  ▼                                             │
│  ┌──────────────────────────────────────────────┐              │
│  │  Gateway node                                │              │
│  │  (OpenWRT router primary; Mac for dev only)  │              │
│  │  ├─ Mesh discovery                           │              │
│  │  ├─ WireGuard inbound (LAN hop, blind) — or  │              │
│  │  │    legacy SOCKS5 on Mac dev gateway       │              │
│  │  └─ Forwards opaque Reality bytes to exit    │              │
│  └──────────────────────┬───────────────────────┘              │
│                         │                                      │
└─────────────────────────┼──────────────────────────────────────┘
                          │ TLS over :443, SNI masqueraded as
                          │ a permitted destination domain
                          ▼
              ┌─────────────────────────────┐
              │  Exit server (VPS)          │
              │  ├─ Xray VLESS+Reality      │
              │  └─ Subscription auth       │
              └─────────────┬───────────────┘
                            │
                            ▼
                      Open internet
```

### 4.2 Node roles

- **Client.** A user's phone (Android 10+, iOS 11+). Discovers nearby gateways, picks the nearest one, tunnels traffic through it. Phones never participate in routing the traffic of *other* users — they are leaf nodes by design.
- **Gateway.** A device with reliable internet upstream (outside the censored region, ideally) and the ability to accept inbound LAN connections. Runs the mesh protocol, accepts WireGuard tunnels from clients on the LAN, forwards opaque Reality bytes to an exit server. The production target is an **OpenWRT router** (kernel WireGuard, low-power, always-on); a Mac app exists for development; a Python reference daemon implements the same protocol for Linux VMs and OpenWRT.
- **Exit server.** A VPS running VLESS+Reality. Current implementation uses a single operator-controlled VPS; production design requires multiple exits in multiple jurisdictions (see roadmap).
- **Bootstrap service.** A CDN-fronted HTTPS service that the mobile app contacts on first launch to learn about available exit servers. Written but not yet deployed.

**Why iPhones are client-only — permanently.** Apple's NetworkExtension sandbox blocks `listen()` and reliable background networking required for the gateway role. This is an OS architectural constraint, not a missing entitlement or a future enhancement. iPhones run mesh discovery and the H4c WireGuard outbound; they never accept inbound LAN connections from other users. This is decided and stable.

**Why Android phone-as-gateway is a Phase 3 fallback, not the primary path.** Android can technically host an inbound WG endpoint via a userspace engine, but battery, thermal, single-radio, and carrier-tethering policy realities make it a poor permanent gateway. Android phones can opt into an *ephemeral hotspot egress* mode for outage scenarios — useful when the local OpenWRT/Starlink dies and a volunteer flips their phone to hotspot to keep the cell online. This is a deliberate optional feature, not the deployment story.

### 4.3 Platform support matrix

| Platform | Client role | Gateway role |
|----------|-------------|--------------|
| Android 10+ | ✅ Full (foreground + background mesh) | ⚠️ Phase 3 ephemeral-hotspot fallback only — not the production path |
| Android 6–9 | ⚠️ Client only, manual WiFi join (OS restriction) | Not supported |
| iOS 11+ | ✅ Foreground client | ❌ **Never** — Apple NE-sandbox blocks `listen()`; architectural constraint, not a workaround target |
| macOS | ✅ (via the Mac app as a client too) | ✅ Development gateway only — not for production |
| OpenWRT routers (kernel WireGuard) | Not applicable | 🎯 **Primary production gateway** — daemon scaffolded (Python reference implementation), `.ipk` package in M1 |

---

## 5. Zero-Touch Onboarding

A core UX goal: a user with a blocked phone opens OpenBand and is online, with no passwords, QR codes, or technical configuration.

### 5.1 Credential derivation

The gateway's WiFi SSID and password are both derived from a **shared salt + UTC date** using SHA-256:

```
hash     = SHA-256(salt + ":" + "YYYY-MM-DD")
SSID     = "OB-" + first 8 hex chars of hash     # e.g. "OB-40f05ec8"
password = next 16 hex chars of hash              # e.g. "cb0a918522114973"
```

Every OpenBand node — mobile app, Mac app, future router — computes the same SSID/password pair for a given UTC date. Credentials rotate at UTC midnight.

This is implemented identically in three languages:
- Swift (`CredentialGenerator.swift`) on macOS and iOS
- Kotlin (`CredentialGenerator.kt`) on Android

**Current limitation:** the shared salt is compiled into every binary. An attacker who reverse-engineers the APK can derive tomorrow's credentials. This is documented as a pre-beta blocker — the salt must be moved server-side (delivered per-subscription over the encrypted tunnel) before production deployment. See Section 9.

### 5.2 Auto-join mechanics

- **Android 10+** uses `WifiNetworkSuggestion`. The app submits today's SSID + password as a network suggestion. The OS displays a one-time notification when the suggested SSID is seen in a scan; after user approval, the phone auto-associates with that SSID whenever it is in range.
- **iOS 11+** uses `NEHotspotConfigurationManager.apply()`. Same UX pattern: the configuration is stored, a prompt appears when the SSID is in range, and the phone auto-associates thereafter.
- **Android 6–9** — auto-join is not possible due to OS restrictions on third-party WiFi manipulation. Users on these devices must join the network manually through system settings.

The point is: users in censored regions do not need to memorize anything. Installing the app is the entire setup.

---

## 6. Mesh Protocol

Once a client is on the same LAN as a gateway, they discover each other and elect a relay.

### 6.1 Transport

- **Protocol:** UDP
- **Port:** `5555`
- **Encoding:** UTF-8 JSON, one message per datagram, <1200 bytes
- **Distribution:** kernel-computed subnet broadcast + `255.255.255.255` (broadcast-as-a-service)

### 6.2 Message types

| Type | Purpose | Frequency |
|------|---------|-----------|
| `heartbeat` | Announce presence + metrics | Every 2 s |
| `keepalive` | Current gateway confirms it is alive | Every 5 s, unicast to peers |
| `elect` | Candidate broadcasts its score during an election round | On election trigger, rate-limited |
| `gateway_announce` | Elected gateway declares itself | Every 2 s + on event |
| `ack` | Confirms delivery of election / announce | Reactive |

### 6.3 Election scoring

Each node computes its own score:

```
score = 0.4 · rssi_norm + 0.3 · (1 / hops) + 0.3 · battery_norm
```

- `rssi_norm` normalizes WiFi RSSI into [0, 1]
- `hops` counts mesh hops to an internet-capable node
- `battery_norm` normalizes battery percentage

Gateway nodes (Mac, router) hardcode `score = 1.0` — they are plugged in, have wired internet, and sit at zero hops from the exit. The highest-scoring candidate wins.

**Tie-breaker:** lexicographically smaller `node_id` wins. Deterministic.

**Election window:** 300 ms. A candidate broadcasts its score, collects peer scores for 300 ms, then resolves.

### 6.4 Election state machine (client side)

```
Idle ── gateway_announce received ──→ Connected to G
Connected ── no keepalive for 5 s ──→ Election in progress
Election ── peer sends higher score ──→ Yield, become Client
Election ── 300 ms elapsed, local score highest ──→ Declare self as gateway
```

Phones are implicitly expected to **yield** to a Mac or router scoring 1.0 — the formula ensures a Mac always wins in any realistic deployment.

### 6.5 Current protocol limitations

- **Packets are unencrypted.** Anyone on the same LAN can observe the full mesh topology (node IDs, neighbor lists, gateway IPs). Planned v2 wraps packets in ChaCha20-Poly1305 with a PSK derived from the daily credential.
- **No authentication.** Anyone on the same LAN can inject fake `gateway_announce` messages. This is mitigated in practice by the Reality handshake on the exit (a rogue gateway can't forge the exit server's key), but mesh-level spoofing is not prevented.
- **Election storms.** Rate-limited but not robust under heavy churn.

---

## 7. Gateway Trust Model

The single most important design decision in OpenBand is: **we do not trust gateway operators**.

### 7.1 Why

A gateway operator is whoever is running the gateway software — a volunteer, an operator, a state-level adversary, a compromised friend. Protocol design cannot assume their honesty.

### 7.2 Architectural evolution (H1 → H4 → H4c)

The relay path has gone through three iterations, documented honestly:

**H1 — SOCKS5 (shipped April 2026):** the gateway operates a SOCKS5 proxy and parses each `CONNECT <host>:<port>`. Works in cooperative tests, but **the gateway sees every destination a user visits**. This is the legacy path; it survives only on the macOS development gateway.

```
H1: Phone ── SOCKS5 ──→ Gateway ── VLESS+Reality ──→ Exit ── TLS ──→ Destination
                          ↑
                gateway parses destinations
```

**H4 — blind TCP passthrough (attempted, abandoned):** attempt to make the gateway a transparent TCP forwarder so the Reality handshake completes phone-to-exit. Hit a protocol-fingerprint mismatch — the Reality server consistently rejected the forwarded handshake and served its masquerade target, indicating a sensitivity we could not isolate without server-side debugging access. Abandoned April 2026.

**H4c — WireGuard LAN hop with nested Reality (decided 2026-04-22, in progress):** the phone wraps its Reality stream inside a WireGuard tunnel terminated by the gateway. The gateway decrypts the outer WG packets and forwards the still-encrypted inner Reality bytes to the exit. **Reality remains end-to-end phone-to-exit; the gateway sees only opaque bytes.**

```
H4c (target):
Phone ── [WG[ Reality[ app ] ]] ──→ Gateway ── [Reality[ app ]] ──→ Exit
                                       ↑
                          gateway decrypts WG only;
                          inner Reality stays opaque
```

**Why WireGuard for the LAN hop, not VLESS re-termination?** A re-termination design (a brief pivot we considered as "H4b") would have left the gateway with plaintext destination visibility — defeating the privacy goal. WireGuard with nested Reality preserves the blind-gateway property without any new key material on the gateway side beyond standard WG peering.

**What the gateway can and cannot observe under H4c:**

| Gateway observes | Gateway cannot observe |
|------------------|------------------------|
| Timing of connections | Destinations |
| Byte volume per WG peer | TLS SNI |
| The fact that a phone is relaying through it | Inner application protocol |
|                                            | Per-flow Reality handshake details |

### 7.3 Current implementation status

| Component | Status |
|-----------|--------|
| Phone-side WG outbound — Android (Xray) | ✅ Shipped |
| Phone-side WG outbound — iOS (sing-box) | ✅ Shipped |
| `gateway_announce` carries `wg_pubkey` + `wg_port` | ✅ Shipped |
| `peer_register` mesh control message | ✅ Shipped |
| Gateway daemon — Linux/OpenWRT (Python reference implementation) | ✅ Scaffolded; end-to-end test pending |
| OpenWRT `.ipk` package | 🚧 M1 — pending GL.iNet hardware |
| Multi-router mesh (M2) — `babel`/`batman-adv` over inter-router WG | 🚧 Roadmap |
| Token-gated peer auth (C3 subscription PSK) | 🚧 Post-launch |

This is described in more detail in the repository's `GATEWAY_PRIVACY.md` and `ROADMAP.md` § H4c.

### 7.4 Residual risks even in target state

Even with a blind gateway, the adversary controlling a gateway can:

- **Time-correlate** client traffic patterns with public signals (who is online when? what volume?).
- **Selectively drop** connections to specific destinations, appearing as "slow internet" to the user.
- **Fingerprint** clients by WiFi MAC address and DHCP behavior.
- **Upload logs later** — the gateway process cannot be prevented from recording and reporting anything it observes.

Mitigations are partial and layered. The strongest is **onion routing across multiple independent gateways**, planned but not implemented.

---

## 8. Distributed Broker and Onion Routing (Target State)

The long-term security north star is a **Snowflake-style rendezvous broker with multi-hop onion routing**.

### 8.1 Rendezvous broker

Inspired directly by Tor's Snowflake:

- Volunteers register their gateways with a CDN-fronted broker (e.g. a Cloudflare Worker).
- Clients authenticate to the broker via a signed subscription token.
- The broker matches one client to one gateway and returns a signed WebRTC session description.
- Clients establish a direct DTLS-encrypted WebRTC DataChannel to the gateway — no shared knowledge of each other's IP until the handshake.
- The broker does not store client↔gateway mappings after matching.

**Clients never learn gateway IPs directly.** A rogue client cannot enumerate gateway IPs by running queries.

### 8.2 Multi-hop (onion)

Phone → Gateway₁ → Gateway₂ → Exit. Each hop decrypts its own layer only. No single gateway sees `(user, destination)` as a pair. This is Tor's core guarantee.

### 8.3 Properties this provides

- Adversary running gateways learns only their own hop's metadata.
- Active IP enumeration of the gateway pool requires many compromised clients and is rate-limited by the broker.
- Adds **user-identity unlinkability** on top of the destination unlinkability that H4c already provides — under H4c a single gateway already cannot see destinations; multi-hop additionally ensures that the gateway *adjacent to the exit* cannot see *which user* is making the request.

### 8.4 Implementation cost

- Broker MVP (Cloudflare Worker) — approximately one engineering week
- WebRTC integration on RN clients — approximately one week
- Multi-gateway onion logic — approximately two to three weeks
- Per-user subscription tokens — approximately one week (ties in with subscription infrastructure)

Total estimate: ~4–6 weeks to a defensible multi-hop design. This is the v2.0 target.

---

## 9. Security Analysis — Current Posture

This section is written with deliberate transparency. OpenBand is **pre-beta**. We describe what is protected today, what is not, and what is planned.

### 9.1 What is protected today

- ✅ **Exit traffic content** — VLESS+Reality between the gateway and the exit server hides content from any network observer between them.
- ✅ **Destination IP from DPI** — the TLS SNI masquerades as a permitted high-trust domain; the apparent destination from the DPI's perspective is that domain.
- ✅ **Mesh topology from off-network observers** — UDP broadcast doesn't cross LAN boundaries, so only attackers on the same LAN can see it.

### 9.2 What is not protected today (pre-beta blockers)

The following are all documented in `SECURITY.md` in the repository as pre-beta blockers:

1. **Hardcoded exit server IP in the binary.** A single block-list addition kills the service for all users until an app update ships. Mitigation plan in `EXIT_INFRASTRUCTURE.md`: dynamic pool of 5–7 servers across 5+ providers, discovered via domain-fronted bootstrap with DoH fallback.
2. **Shared VLESS UUID.** Reverse-engineered from the APK, a single leaked identifier can be fingerprinted and blocked. Mitigation: per-user UUIDs issued after on-chain USDC subscription (or during a blackout-relief window, no subscription required); revocable per-user.
3. **Salt compiled into the binary.** Anyone with the APK computes tomorrow's SSID and password. This defeats the purpose of daily rotation. Mitigation: salt moves server-side, delivered per-subscription over the encrypted tunnel.
4. **Single exit server today (strategy already defined).** Single-point-of-failure against IP-level blocking. See `EXIT_INFRASTRUCTURE.md` for the full multi-server strategy: geographic + jurisdictional + provider + ASN + IP-range diversity, automated rotation, and health monitoring. Target: 5–7 concurrent exits.
5. **Mesh packets unencrypted.** Any attacker on the same LAN captures the topology. Mitigation: v2 of the protocol wraps each datagram in ChaCha20-Poly1305 with a PSK derived from the daily credential.
6. **No internet-reachability check in the election.** A node on a captive portal can win the election and black-hole traffic. Mitigation: reachability probe multiplied into the election score.
7. **Gateway sees destinations under the legacy H1 SOCKS5 path.** Section 7.2 above. **Mitigation: H4c — WireGuard LAN hop with nested Reality end-to-end (in progress).** Phone-side shipped on Android (Xray) and iOS (sing-box); Linux/OpenWRT gateway daemon scaffolded; end-to-end validation pending. Snowflake-style broker + onion routing (Section 8) is the longer-term layered defense for additional unlinkability.

None are mitigated today. Beta release blocks on all seven; most have concrete implementation plans in the referenced docs.

### 9.3 Residual risks even after all pre-beta fixes

Even after dynamic server config, per-user UUIDs, server-side salts, multiple exits, mesh encryption, and reachability checks, the adversary retains the following:

- **Timing/volume correlation** across the relay chain.
- **Client WiFi fingerprint** (MAC address, probe request patterns).
- **Seizure of a user device** — full profile extraction for that user.
- **Compromise of broker infrastructure** — enumeration of gateway pool at a point in time.

The planned multi-hop onion routing (Section 8) addresses most of these — not perfectly, but materially.

### 9.4 Cryptographic primitives

- **Exit:** VLESS + Reality over TLS 1.3. Curve25519 for Reality proof-of-knowledge, X25519 for TLS handshake, AES-GCM or ChaCha20-Poly1305 for record encryption.
- **Mesh (planned v2):** ChaCha20-Poly1305 with a PSK derived from the daily credential.
- **Phone↔gateway LAN hop (H4c, in progress):** WireGuard (Noise_IKpsk2 — ChaCha20-Poly1305, Curve25519, BLAKE2s). Reality remains end-to-end inside the WG tunnel; gateway sees only opaque encrypted bytes. See `GATEWAY_PRIVACY.md`.
- **Inter-router mesh (M2):** WireGuard between adjacent routers, with `babel` (or `batman-adv`) running on top for path selection. Reality remains end-to-end across all hops.
- **Broker (planned):** Ed25519 signatures on broker responses, DTLS 1.3 on the resulting WebRTC channel.

---

## 10. Operational Considerations

### 10.1 Running an exit server

A minimum exit is a VPS with a public IPv4, running Xray with a VLESS+Reality inbound on port 443, SNI masqueraded to a high-trust domain (configurable per operator). Recommended: non-Five-Eyes jurisdiction, unattributed payment, rotated IP every 30–90 days.

The monthly cost of a single exit is under USD 10. **A production deployment requires 5–7 exits across different hosting providers, legal jurisdictions, ASNs, and IP ranges**, rotated on a quarterly schedule and on-demand when blocked. The detailed strategy — diversity matrix, Terraform/Ansible provisioning, health monitoring, incident response — is in `EXIT_INFRASTRUCTURE.md`.

Baseline operational cost for a multi-exit deployment is approximately USD 40–80 per month. Break-even at ~40 paying subscribers at USD 2/month.

### 10.2 Running a gateway

The Mac app is the reference gateway implementation. A gateway node needs:

- Stable internet connection in an uncensored jurisdiction
- WiFi broadcasting capability (or Ethernet downstream to a router)
- Approximately 50 MB of RAM for the mesh discovery and relay stack

The OpenWRT target is a GL.iNet GL-MT300N-V2 (≈USD 30), producing a pocket-sized always-on gateway suitable for shipping pre-configured to users.

### 10.3 Bandwidth economics

Per-user bandwidth is dominated by the user's own consumption plus a small mesh overhead (≈2 kbps). A volunteer gateway with a 1 Gbps residential link can support dozens of concurrent users without saturation. The limiting factor in practice is usually the exit server's egress.

### 10.4 Trust bootstrapping

Users acquire the mobile app via an out-of-band channel: TestFlight for iOS (during beta), signed APK distributed via Telegram groups, or eventually Play Store / App Store if the jurisdiction allows. The very first launch contacts a domain-fronted bootstrap service to learn about available exits. This bootstrap service — not the app binary — is what requires dynamic updating as IPs are blocked.

---

## 11. Governance, Economics, and the Blackout Clause

OpenBand is structured as a **volunteer-operated network with a small operator core**. Every component that needs funding is deliberately designed so that funding can come from multiple independent sources — grants, donations, subscriptions, or operator contributions — rather than a single monolithic revenue stream.

### 11.1 Current phase — team-funded, pre-launch

OpenBand today is funded entirely by the **founding team, as volunteers**. There is no donation intake in operation yet, no subscription revenue, no investor funding, and no token. The core team pays the exit-server and bootstrap-infrastructure costs out of pocket while the system matures to the point where it can responsibly accept outside support.

This phase is deliberately self-constrained. Accepting donations or subscription payments before the security posture can protect paying users at scale would be a mis-step. When the pre-beta blockers in § 12 are closed, the funding model transitions to the post-launch structure below.

### 11.1.1 Planned funding intakes (post-launch)

Two independent intakes, both denominated in **USDC on Base** (Coinbase's Ethereum L2):

1. **Subscriptions** from users in free-internet jurisdictions (see § 11.2 for the detailed terms). Recurring, opt-in, ~USD 2–5 per month.
2. **Donations earmarked for censored-territory service.** Donors anywhere can contribute to the pool that pays for free access in regions where OpenBand is offered free by principle (see § 11.3). On-chain, traceable, auditable.

Post-launch, both intakes fund the same three cost centers:

- **Exit-server hosting** (5–7 nodes across neutral jurisdictions).
- **Gateway operator rewards** — volunteers running OpenBand gateways receive USDC payouts proportional to bandwidth served and uptime. Rewards are on-chain, anonymous, and do not require operators to disclose identity beyond what Base itself requires.
- **Starlink / satellite-uplink reimbursement** — operators whose gateway is backhauled over a satellite constellation have their monthly subscription fee reimbursed from the pool, making satellite-backed gateways economically viable for volunteers.

Donation wallet addresses will be published through project channels and verifiable on-chain. All disbursements to operators will be traceable and auditable by any donor.

### 11.2 Post-launch — USDC-on-Base subscriptions

When OpenBand exits pre-beta, end-user subscriptions will be paid directly in **USDC on Base**:

- **Why Base:** Ethereum L2 with sub-cent transaction fees, native USDC support (Circle's official deployment), trivial fiat on-ramp through Coinbase Commerce for non-crypto users.
- **Why USDC:** dollar-pegged stablecoin, regulatory clarity in most jurisdictions, zero price-volatility risk for users topping up subscriptions.
- **What replaces the corporate billing paper trail:** subscriptions are recorded as on-chain events associated with a pseudonymous subscriber key (not a real-world identity). The operator sees "an anonymous subscriber paid X USDC for Y days of service" — no KYC, no credit card, no address.
- **Target pricing:** USD 2 per month (conservative) to USD 5 per month (comfortable) for unlimited bandwidth. Lower than commercial VPNs because the infrastructure is distributed rather than centrally operated.

TON and other alternatives were evaluated and set aside: Base/USDC has better fiat-ramp UX for end users, wider developer tooling, and lower operational friction for the project operators.

### 11.3 The blackout clause — free access during state-level outages

**When a target region experiences a state-level internet blackout or major circumvention-tool ban, OpenBand is free for users in that region.** Subscriptions are suspended for the duration of the event and a cooldown period after.

This is a humanitarian commitment, not a marketing promise. When people in any target jurisdiction are cut off from the internet during a political event, cost is not the first barrier we want them to encounter.

Operationally, blackouts are detected by cross-referencing multiple independent monitors:

- **[OONI](https://ooni.org/)** — Open Observatory of Network Interference.
- **[IODA](https://ioda.inetintel.cc.gatech.edu/)** — Internet Outage Detection and Analysis.
- **[Cloudflare Radar](https://radar.cloudflare.com/)** — real-time traffic anomaly detection.
- **Kentik**, **Censys**, and on-the-ground operator reports.

When two or more independent signals agree that a region is experiencing a major outage, the project's on-chain contract automatically waives subscriptions for that region's subscribers. During the blackout window, operator rewards continue to be paid from the earmarked-donations pool (and, during the team-funded phase, from the team's own runway) to ensure the network stays up precisely when users need it most.

### 11.4 Roles in the network

- **Subscribers** — end users in censored regions. Pay subscriptions in USDC on Base (or are free during blackouts). Anonymous.
- **Gateway operators** — volunteers running OpenBand gateway software on a Mac, OpenWRT router, or Starlink-connected device. Earn USDC rewards proportional to traffic served.
- **Exit-server operators** — run VLESS+Reality exit servers in neutral jurisdictions. Paid from subscription + donation pools.
- **Bootstrap operators** — run CDN-fronted bootstrap services that clients contact on first launch. Low cost, operator-funded or donation-funded.
- **Donors** — post-launch, fund the network by contributing to the earmarked pool that pays for free access in censored territories and blackout-relief. (No donation intake in operation yet — the team self-funds until post-launch.)

The target scale is **thousands of active gateway nodes** to make enumeration economically prohibitive for any state adversary.

### 11.5 What we're deliberately not doing

- **No ICO, no token launch, no governance token.** OpenBand uses USDC because it is fit-for-purpose (stable, liquid, widely understood). A new project-specific token would add speculative risk, regulatory complexity, and governance overhead with no compensating benefit for the core mission.
- **No user data sales.** Ever.
- **No advertising.** Ever.
- **No partnerships with data brokers or "insight platforms."** Ever.
- **No source-code publication during pre-launch.** The implementation is proprietary. We have chosen to lean on the public threat model, public protocol specifications, and (post-launch) independent audits for trust — not on source availability. This is a deliberate tradeoff: we believe copy-risk outweighs the incremental trust gained from publishing code *before* the security posture hardens. Post-launch we may migrate to a source-available license; see `LICENSE` in the public docs repository.

Revenue exists to keep the network alive. That is the entirety of the business model.

---

## 12. Implementation Status

We report implementation status against a simple rubric: **works in a real test with hostile-equivalent conditions**, **works in cooperative test**, or **not implemented**.

| Component | Status | Notes |
|-----------|--------|-------|
| Mesh discovery (3 platforms) | ✅ Works in cooperative test | Subnet broadcast; election stable |
| Gateway election (single-cell, single-hop) | ✅ Works in cooperative test | Mac wins predictably; dissolves under multi-router mesh (M2) into routing-protocol path selection |
| SOCKS5 relay through gateway (H1) | ✅ Works in cooperative test | Real web traffic verified; legacy Mac dev path |
| Phone-side H4c WG outbound (Android + iOS) | ✅ Builds and integrates | End-to-end blind-gateway test pending |
| Linux/OpenWRT gateway daemon | ✅ Scaffolded | Awaiting test hardware or VM validation |
| VLESS+Reality exit | ✅ Works | Production-grade protocol |
| Zero-touch onboarding (Android 10+, iOS 11+) | ✅ Works in cooperative test | H5 shipped |
| Blind gateway via H4 (transparent TCP passthrough) | ❌ Attempted and failed (April 2026) | Reality handshake rejected by exit — abandoned |
| Blind gateway via H4c (WireGuard LAN hop, nested Reality) | 🚧 Phone-side shipped; gateway daemon scaffolded; end-to-end test pending | Replaces both H4 and the considered-but-unimplemented H4b VLESS re-termination |
| Dynamic exit server configuration | ❌ Not implemented | Pre-beta blocker #1 |
| Per-user tokens | ❌ Not implemented | Pre-beta blocker #2 |
| Server-side salt | ❌ Not implemented | Pre-beta blocker #3 |
| Multiple exit servers | ❌ Only one | Pre-beta blocker #4 |
| Mesh encryption | ❌ Plaintext JSON | Pre-beta blocker #5 |
| Internet reachability scoring | ❌ Not implemented | Pre-beta blocker #6 |
| Bootstrap broker deployed | 🚧 Written, not deployed | |
| Multi-hop onion routing | ❌ Not implemented | v2.0 north star |
| Snowflake-style WebRTC broker | ❌ Not implemented | v2.0 north star |

A publishable v1.0 beta requires the seven pre-beta blockers to be closed. Our internal estimate is one to two months of focused engineering from the current state.

---

## 13. Why Publish This Document Now

One legitimate critique: publishing a whitepaper for a tool the state wants to block seems counterproductive. We think the opposite, for three reasons:

1. **The adversary already knows the broad approach.** Tor, Snowflake, Psiphon, Lantern, and others have publicly documented similar designs for years, and state-level censors have invested heavily in countering them. Keeping *our* specific architecture private does not materially disadvantage an adversary who has studied the field.
2. **Security through obscurity is not a defense.** Every meaningful security property of OpenBand comes from cryptographic primitives and protocol design, not secrecy of design. If someone can defeat us by reading this document, the design is insufficient regardless.
3. **Public review improves the system.** Anti-censorship tools that have been peer-reviewed by the security community (Tor, Signal, Tails) are more robust than tools built behind closed doors. We invite scrutiny.

We are explicit that this document describes a *pre-beta* system. None of the security claims constitute production guarantees. We will revise this document at each beta release as more properties move from "planned" to "shipped."

---

## 14. How to Contribute

- **Audit the design.** The repository is structured to make architectural claims verifiable. Start with `docs/ARCHITECTURE.md`, `docs/SECURITY.md`, and `docs/GATEWAY_PRIVACY.md`.
- **Run a gateway.** When gateway software is ready for volunteer operators (target: early beta), running one is the highest-leverage contribution to the network's resilience.
- **Adversarial review.** If you can break the design, please do — privately first, then publicly after a fix window.
- **Operational support.** Sysadmin / SRE help for exit-server rotation is always welcome.

Security issues: contact the project privately. Non-security contributions: standard PR / issue workflow in the public repository once it is open.

---

## 15. Acknowledgements

OpenBand builds on decades of work by anti-censorship researchers and engineers: the Tor Project, Xray and sing-box contributors, the Signal team, the Snowflake and meek authors, and every volunteer who has ever run a public relay. The WireGuard, uTLS, and Network.framework projects provide the low-level primitives. The design mistakes are ours; the primitives are theirs.

---

## 16. References and Further Reading

- **Tor Project** — https://www.torproject.org
- **Snowflake (Tor)** — https://snowflake.torproject.org — foundational for our gateway-privacy plan
- **Reality / VLESS / Xray** — https://github.com/XTLS/Xray-core
- **sing-box** — https://sing-box.sagernet.org
- **WireGuard** — https://www.wireguard.com
- **uTLS** — https://github.com/refraction-networking/utls
- **Psiphon** — https://psiphon.ca — multi-protocol obfuscation
- **Briar** — https://briarproject.org — local mesh messaging precedent
- **Apple NetworkExtension** — https://developer.apple.com/documentation/networkextension
- **Android WifiNetworkSuggestion** — https://developer.android.com/guide/topics/connectivity/wifi-suggest

Repository-internal references (for readers with access):

- `docs/ARCHITECTURE.md` — full system diagram and data flow
- `docs/MESH_PROTOCOL.md` — on-wire protocol specification
- `docs/SECURITY.md` — threat model and residual risk analysis
- `docs/GATEWAY_PRIVACY.md` — blind-gateway and broker designs
- `docs/ROADMAP.md` — priority-ordered engineering plan

---

*Document version 0.1, April 2026. Next revision planned at first beta cut. Feedback and critique welcome.*
