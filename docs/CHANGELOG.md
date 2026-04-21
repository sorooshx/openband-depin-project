# Changelog

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versions track chat-sprint work rather than semver until v1.0.

## [Unreleased]

### Added (2026-04-20 — doc + H1 session + architectural pivot)
- Full docs suite under `docs/`: PROJECT_STRUCTURE, ARCHITECTURE, RUNBOOK, MESH_PROTOCOL, SECURITY, DEV_ENVIRONMENT, ROADMAP, CHANGELOG, CONTRIBUTING, GATEWAY_PRIVACY, README.
- GATEWAY_PRIVACY.md: Snowflake-style rendezvous broker as v2.0 architectural north star (C6 in roadmap).
- ARCHITECTURE.md: "Platform roles" matrix locking in iOS as client-only. Later: new "Trust model" section for blind-gateway pivot, new system diagram showing end-to-end Reality via passthrough.
- RUNBOOK.md: "Mac as test router" section (Mode A: same WiFi, Mode B: hotspot).
- SECURITY.md: new **"Gateway-as-adversary threat model"** section with 5 residual risks (A–E) and mitigations for each.
- ROADMAP.md: H4 (Blind gateway TCP passthrough) + H5 (Zero-touch onboarding with Reality validation).
- iOS JS glue for H1 traffic relay: `VpnService.ts` now bridges DiscoveryModule events → `VpnModule.setGateway()`.
- `setGateway:` exposed in `VpnModule.m` ObjC bridge.
- Mac `MeshDiscovery.swift`: periodic `gateway_announce` timer (2s) + TX logging; subnet broadcast now reads kernel-computed address from `getifaddrs` (fixed /22 networks); sends to both subnet + `255.255.255.255`.
- iOS/Android `handleElect`: guard against fresh election when a live gateway already exists (prevents election loop that kept iPhones self-electing).
- Android `XrayConfigBuilder.kt`: routing rules no longer require `geoip.dat` (was crashing Xray on every launch with `geoip:private`); explicit RFC 1918 CIDRs used instead. Log level lowered to `error`.
- Android `XrayVpnService.kt`: TUN routes now exclude all private LAN, loopback, link-local, multicast, broadcast — fixes feedback loop where Xray's own outbound to Mac was being captured by TUN and looping back into Xray.
- Android `DiscoveryForegroundService.kt`: `onGatewayChanged` only restarts Xray when endpoint actually changes (was churning every 2s with Mac's re-announces).

### Changed (2026-04-20)
- iOS background VPN and iOS-as-gateway formally dropped (previously "parked"). See ROADMAP scope decisions.
- **Architectural pivot**: gateway redesigned from SOCKS5 proxy to **blind TCP passthrough**. Phone's VLESS+Reality handshake is end-to-end with Hetzner. Gateway sees only encrypted bytes. Fixes the largest rogue-gateway surveillance surface. H1 work superseded by H4; election + hot-swap plumbing reused.

### Fixed (2026-04-20 debugging session)
- Mac mesh broadcast was sending to wrong address on `/22` networks (computed `.X.255` assuming /24). Now uses kernel-reported broadcast.
- Mac only sent `gateway_announce` reactively — phones never heard it reliably. Now broadcasts every 2s.
- iPhone couldn't receive broadcasts despite entitlement in source: provisioning profile was the old wildcard. User refreshed profile in Xcode after Apple approval; phones now see Mac.
- Xray crash on every start due to missing `geoip.dat` — replaced with explicit private CIDRs.
- TUN self-loop: Xray's SOCKS5 outbound to Mac was being captured by its own TUN. Exclude private LAN from TUN routes.
- Xray config churn: `restartXrayWithConfig` was firing every 2s on every re-announce. Now only on endpoint change.

### Planned
- **H4 — Blind TCP passthrough gateway** (~2 days, highest priority)
- **H5 — Zero-touch gateway auto-discovery + Reality validation** (~4 days)
- Dynamic server config fetch (C1)
- Bootstrap server deployed to Hetzner :3210 (H3)
- ChaCha20-Poly1305 on mesh UDP (C4)
- Per-user VLESS UUIDs via Telegram bot (C3)

---

## [Chat 09] — 2026-03-26 — Mac gateway + bootstrap server

### Added
- **Mac app** (`~/OpenBandMac/`) as gateway node running sing-box, mesh discovery, Internet Sharing.
- `GatewayManager.swift` orchestrator for Mac app.
- `MeshDiscovery.swift` on Mac, shares the same JSON/UDP :5555 wire format as mobile.
- `SingBoxManager.swift` spawns sing-box subprocess and exposes SOCKS5 on :10808.
- `BootstrapClient.swift` (Mac) + `bootstrap/server.js` (Node.js/Express) for rendezvous.
- `CredentialGenerator.swift` for daily rotating SSID/password via SHA256(salt:date).
- HomeScreen, NodesScreen, AccountScreen in Mac app.

### Changed
- Election scoring now has Mac hardcoded at 1.0.
- Rate limiting applied per message type to prevent ACK storms.

### Known issues
- Bug #1 election override (fix written, unverified).
- Bug #4 traffic relay not wired — phones elect Mac but don't route through it yet.
- Bootstrap server written but not deployed.

### Commits
- `cbacab8` Update Android VPN manager and nodescreen

---

## [Chat 08] — Pending — Telegram bot + USDC-on-Base subscription
Pending. Ties into C3 (per-user VLESS UUID). Economic model decision 2026-04-21: subscriptions paid in USDC on Base (not TON). Free during confirmed internet blackouts. Gateway-operator rewards + Starlink reimbursements funded from donations today, subscriptions post-launch.

---

## [Chat 07] — Done — Phase 3: ACK + failover + election

### Added
- ACK message type on gateway_announce / elect.
- Failover on 15s gateway silence.
- Election scoring formula: `0.4*RSSI + 0.3*(1/hops) + 0.3*battery`.

### Fixed
- Mesh discovery no longer flaps on brief WiFi drops.

---

## [Chat 06] — Parked — iOS background VPN

### Attempted
- WireGuard for iOS implementation (`9200cbd`, `7112a6d` — failed).
- iOS VPN debugging across multiple sub-chats (`abe90a2`, `0ae13e5`, `eed105c`, `4e7ad08`).

### Outcome
- Parked. NE extension sandbox blocks `listen()`. `includeAllNetworks` entitlement missing.
- Next viable path: WireGuardKit via SPM in PacketTunnelProvider.

### Commits
- `9200cbd` WireGuard for ios implementation
- `7112a6d` Ios VPN with wireguard failed
- `eed105c` iOS VPN debugging moved to Chat 6-3
- `4e7ad08` iOS VPN debugging - Finish Chat6-3

---

## [Chat 05] — Pending — Website openband.io on Vercel
Pending. Marketing + download page.

---

## [Chat 04] — Done — gomobile bind + iOS VPN foreground

### Added
- `Mobile.xcframework` built via `gomobile bind -target ios/arm64` from sing-box fork.
- iOS VPN foreground mode (sing-box running in main app process, SOCKS5 :10808 / HTTP :10809).
- HTTP proxy inbound on iOS for local testing.

---

## [Chat 03] — Done — NEPacketTunnelProvider + Xray arm64

### Added
- `PacketTunnel/` NE extension target.
- `PacketTunnelProvider.swift` proxy-only mode.
- Xray arm64 binary integration.

---

## [Chat 02] — Done — iOS DiscoveryModule + bidirectional mesh

### Added
- `DiscoveryModule.swift` on iOS.
- `DiscoveryModuleShared.swift` neighbor state.
- iOS ↔ Android mesh handshake confirmed.

---

## [Chat 01] — Done — Full project setup + Android mesh

### Added
- React Native 0.73.4 scaffold, bundle `com.openband.app`.
- Zustand store (`useVPNStore.ts`).
- HomeScreen, NodesScreen, AccountScreen (initial).
- Android `XrayVpnService`, `MeshController`, `UdpDiscoveryService`.
- Android VPN tunnel confirmed to Hetzner via VLESS+Reality.

### Commits
- `6c3710e` OpenBand v1.0 — Phase 2 complete, Tesla UI
- `8794334` Update The UI
- `05990d9` MESH TESTING PASSED
- `843ac5b` Mesh Network works well

---

## How to update this changelog

When finishing a chat sprint:

1. Move `[Unreleased]` items that shipped into a new `[Chat N]` section with today's date.
2. Group by **Added / Changed / Fixed / Removed / Deprecated / Security / Known issues**.
3. Reference commit SHAs when useful (use `git log --oneline -20` to find them).
4. Update [ROADMAP.md](ROADMAP.md) to move the completed sprint to "Done."
