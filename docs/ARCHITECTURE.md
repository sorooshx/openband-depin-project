# Architecture

**Website:** [https://www.openband.io](https://www.openband.io)

## One-paragraph summary

OpenBand is a mesh VPN: phones discover a nearby gateway over UDP multicast (typically a Mac or OpenWRT router in a region with free internet), establish an end-to-end VLESS+Reality tunnel through that gateway to an exit VPS, and exit to the open internet. The gateway is a **blind TCP passthrough** — it doesn't terminate the phone's encryption, so a rogue gateway can't read destinations or content. If a gateway fails, re-election happens in <5s. The long-term goal is per-user UUIDs, multiple exit servers, zero-touch onboarding, and a Snowflake-style rendezvous broker.

## System diagram

```
┌──────────────────────────────────────────────────────────────────┐
│  LAN / OpenBand WiFi hotspot                                     │
│                                                                  │
│  ┌──────────┐    UDP :5555     ┌──────────┐    UDP :5555        │
│  │  iPhone  │ ←───mesh JSON──→ │  Android │ ←───mesh JSON──→    │
│  │  (iOS)   │                   │          │                    │
│  └────┬─────┘                   └────┬─────┘                    │
│       │   VLESS+Reality to gateway (end-to-end w/ exit node)     │
│       ▼                              ▼                          │
│  ┌─────────────────────────────────────────────┐                │
│  │  Gateway — Mac or OpenWRT router            │                │
│  │  ├─ MeshDiscovery      (UDP :5555)          │                │
│  │  ├─ TCP forwarder → exit node (blind)       │                │
│  │  └─ (optional) HotspotManager               │                │
│  └──────────────────────┬──────────────────────┘                │
│                         │ transparent TCP passthrough            │
└─────────────────────────┼───────────────────────────────────────┘
                          │
                          ▼  (phone's VLESS+Reality handshake
                          │   continues end-to-end, untouched)
              ┌─────────────────────────────┐
              │  Exit VPS                    │
              │  ├─ Xray (VLESS+Reality)    │
              │  └─ Admin panel (out-of-band)│
              └─────────────────────────────┘
                          │
                          ▼
                    Open internet
```

The gateway is **end-to-end blind** — it sees only encrypted Reality bytes flowing between the phone and the exit node. No SOCKS5, no destination inspection, no traffic content visibility.

## Components

### Mobile app (React Native)

- **UI layer** — `src/screens/*.tsx`, Zustand store in `src/store/useVPNStore.ts`.
- **JS↔native bridge** — `src/services/VpnService.ts` subscribes to `gatewayChanged`, `becomeGateway`, `gatewayLost`, `vpnStateChanged` events.
- **iOS native** (Swift) — **client-only role**, see [Platform roles](#platform-roles) below:
  - `DiscoveryModule.swift` — mesh discovery, election, publishes RN events. Runs only while app is foregrounded.
  - `VPNManager.swift` + `VpnModule.swift` — sing-box foreground VPN (works) and NE config plumbing.
  - `PacketTunnelProvider.swift` — NE extension, proxy-only. Background VPN has been formally dropped (see [ROADMAP.md § Scope decisions](ROADMAP.md#scope-decisions--ios-role)).
- **Android native** (Kotlin):
  - `XrayVpnService.kt` — VpnService, TUN interface, Xray process.
  - `DiscoveryForegroundService.kt` — keeps discovery alive in background.
  - `MeshController.kt` + `MeshProtocol.kt` — election logic, wire format.
  - `XrayConfigBuilder.kt` — builds outbound config. Direct mode: VLESS→exit node. Relay mode: VLESS→gateway_ip (same Reality keys, gateway forwards transparently).

### Mac gateway app (SwiftUI)

- **GatewayManager** — orchestrator. Owns lifecycle of SingBoxManager, MeshDiscovery, HotspotManager.
- **MeshDiscovery** — speaks the same JSON/UDP :5555 protocol as mobile.
- **TCP Forwarder** (to be added as part of blind-gateway pivot) — listens on the standard HTTPS port, pipes bytes transparently to the exit node. This is the new data path. Gateway sees only encrypted Reality bytes.
- **SingBoxManager** — legacy role (local SOCKS5) kept for Mac's own browsing needs but not used by relayed phones.
- **HotspotManager** — toggles macOS Internet Sharing (limited: needs Ethernet upstream for WiFi sharing).
- **BootstrapClient** — posts node registration, fetches peer list (when bootstrap server deploys).
- **CredentialGenerator** — a keyed daily hash derives the SSID and password for the hotspot; rotates at UTC midnight. Salt and exact derivation are not documented publicly. Likely simplified or removed once zero-touch Reality-validated onboarding lands (see [ROADMAP H5](ROADMAP.md)).

### Exit server

- **Xray** — VLESS+Reality inbound on the standard HTTPS port; SNI masquerade target rotates and is configured per-deployment.
- **Admin panel** — access is out-of-band, not exposed on the public diagram.
- **Bootstrap (planned)** — Express server, see `bootstrap/server.js`.

## Platform roles

Not every platform plays every role. This is a **deliberate scope decision** as of 2026-04-20, locked in after multiple iOS NE-sandbox attempts. See [ROADMAP.md § Scope decisions](ROADMAP.md#scope-decisions--ios-role) for the full rationale.

| Platform | Client | Mesh member (local discovery) | Gateway / relay |
|----------|--------|-------------------------------|-----------------|
| **Android** | ✅ foreground + background | ✅ (foreground service) | ✅ can win election |
| **iOS** | ✅ foreground only | ✅ foreground only | ❌ **never** (architectural) |
| **macOS** | (n/a — not a user device) | ✅ | ✅ **always wins** (score 1.0) |
| **OpenWRT router** ([M1](ROADMAP.md#m1-openwrt-router-package-chat-10)) | (n/a) | ✅ | ✅ always-on dedicated |

**Why iOS can't be a gateway:**
1. NE extension sandbox blocks `listen()`, so it can't accept mesh connections.
2. `includeAllNetworks` entitlement is required for whole-device VPN and isn't available to third-party devs.
3. iOS firewall blocks non-TLS custom protocols we tested through NE.
4. `NEPacketTunnelProvider` is killed by iOS after brief background periods, breaking continuity.

**What iOS *does* do well:** foreground VPN via sing-box + mesh discovery + election participation as a *client*. UX model is identical to commercial VPN apps (NordVPN, ExpressVPN, Proton): tap Connect → traffic routes → backgrounding suspends after ~3 min → reopen to resume. Acceptable for MVP.

**Future: [C6 Snowflake broker](GATEWAY_PRIVACY.md)** gives iOS a better client path via WebRTC DataChannel (native to iOS, no NE constraints), sidestepping the background-VPN problem entirely.

## Data flow: connect button press → exit traffic

**Architectural pivot (2026-04-20, finalized 2026-04-22 as H4c):** the gateway is a **blind byte-pipe** for the VLESS+Reality session that the phone holds end-to-end with the exit node. The phone↔gateway LAN hop is wrapped in **WireGuard** (H4c); the gateway decrypts the outer WG only and forwards the still-encrypted inner Reality bytes. The gateway never sees destinations or content. See [Trust model](#trust-model) below.

The earlier H4 attempt — a transparent TCP passthrough at the gateway with no LAN-hop encryption — was abandoned (Reality handshake rejected by the exit due to a fingerprint sensitivity we could not isolate). H4c (WireGuard outer, Reality inner) replaces it and additionally encrypts the LAN hop. Legacy SOCKS5 (H1) survives only on the macOS dev gateway.

1. User taps **Connect** in HomeScreen.
2. `useVPNStore.connect()` → `VpnService.start()` → native `VpnModule.start()`.
3. Native side spins up TUN + tun2socks + Xray (Android) / sing-box (iOS).
4. Initial config = direct VLESS+Reality to the exit (works without any gateway peer).
5. Concurrently, `DiscoveryForegroundService` (Android) / `DiscoveryModule` (iOS) broadcasts heartbeats on UDP :5555.
6. Gateway responds with `gateway_announce` carrying its `wg_pubkey` + `wg_port`.
7. Phone generates/loads its WG keypair (`WireGuardKeyPair`) and unicasts a `peer_register` mesh message to the gateway with its public key.
8. Gateway adds the phone as a WG peer (`wg set wg0 peer <pubkey> allowed-ips <derived-ip>/32`) — derived IP comes from `SHA-256(pubkey)[0]%253+2`, identical on both sides.
9. Phone hot-swaps engine config to `relayConfigViaWg` — VLESS+Reality to the exit, dialed through a WireGuard outbound that peers with the gateway (`dialerProxy: wg-tunnel`).
10. Reality handshake completes end-to-end phone↔exit. Gateway sees only encrypted WG packets (and after WG decryption, only encrypted Reality bytes).
11. User traffic: app → TUN → engine → `[WG[ Reality[ app ] ]]` → gateway → `[Reality[ app ]]` → exit → open internet.

## Trust model

The gateway is **untrusted.** It can be operated by anyone, including an adversary. The security design does not require gateway honesty:

- **Reality handshake** is between phone and exit node directly. Gateway can't impersonate the exit (no private key) and can't read traffic (TLS).
- **Authentication** is via the exit node's Reality public key embedded in the phone binary. Any gateway relaying to the exit is "valid enough" — if the handshake succeeds, the traffic is protected.
- **Integrity** is via Reality's TLS-in-TLS. A rogue gateway trying to MITM fails the handshake and phone rotates to another gateway.

**What a rogue gateway can still see** (residual attack surface): timing, volume, WiFi MAC, mesh `node_id`. See [SECURITY.md § Gateway-as-adversary threat model](SECURITY.md#gateway-as-adversary-threat-model) for the full list and mitigations.

**Why this is better than the earlier SOCKS5 design:** SOCKS5 requires the gateway to know the destination of every connection ("CONNECT twitter.com:443"). A rogue operator would log every site each phone visits. Blind TCP passthrough eliminates this — the gateway sees only encrypted Reality bytes going to one fixed endpoint.

## Threading / runtime model

- **Android** — discovery runs in `DiscoveryForegroundService` (foreground service w/ persistent notification); Xray runs inside `XrayVpnService` (VpnService subclass). Both survive Doze. UDP socket I/O on its own thread.
- **iOS** — `DiscoveryModule` runs in the main app process (not the NE extension). `VpnModule` (foreground sing-box) runs in-process. `PacketTunnelProvider` is sandboxed and **cannot** open listen sockets — this is why mesh discovery can't happen in the NE extension. This is the root reason iOS is client-only; background mesh is not reachable under current iOS APIs.
- **Mac** — `GatewayManager` on main actor; sing-box is a child process, mesh UDP on a dispatch queue.

## Key invariants

- All three platforms serialize the same JSON shape on UDP :5555. Adding a field means updating `MeshProtocol.kt`, `DiscoveryModuleShared.swift`, and `MeshDiscovery.swift` together. See [MESH_PROTOCOL.md](MESH_PROTOCOL.md).
- Election score formula must match across platforms: `0.4*RSSI_norm + 0.3*(1/hops) + 0.3*battery_norm`. Mac hardcodes 1.0.
- `node_id` must be stable across a session (not per-packet). Format: `Mobile_<last4>`, `Mac_<last4>`, etc.
- Gateway announcement wins over local election if remote score > local score. This is the Chat 09 fix that hasn't been verified yet.

## Out of scope (today)

- Cross-LAN mesh (two houses). Requires relay over the exit node or WebRTC STUN — not started.
- Windows/Linux gateway. Only Mac for now; OpenWRT routers planned ([M1](ROADMAP.md#m1-openwrt-router-package-chat-10)).
- **iOS as a gateway or background mesh node.** Permanently dropped; see [Platform roles](#platform-roles).
- IPv6.
- Anti-DPI shaping beyond what VLESS+Reality already provides.
