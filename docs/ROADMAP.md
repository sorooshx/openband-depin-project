# Roadmap

Status as of **2026-04-20** (end of Chat 09).
Priority is strict: do the Critical items before anything in High, etc.

---

## What works today (confirmed)

- ✅ Android VPN tunnel to Hetzner via VLESS+Reality
- ✅ iOS VPN foreground mode (sing-box in main app process)
- ✅ Mesh discovery on all 3 platforms (Android, iOS, Mac)
- ✅ Gateway election — Mac wins with score 1.0 in steady state
- ✅ `gatewayChanged` / `becomeGateway` / `gatewayLost` JS events
- ✅ Mac sing-box tunnel confirmed (`curl --socks5 127.0.0.1:10808 https://api.ipify.org` → `<exit-server-ip>`)
- ✅ Multicast exclusion from Android VPN TUN (no mesh traffic leaks through tunnel)
- ✅ Rate limiting on mesh packets (no ACK storms)
- ✅ Mac Nodes screen shows Android + iOS live

---

## Known bugs

### Bug #1 — Gateway election override (unverified fix)
When Android VPN turns on, iOS connects to Android instead of Mac.
**Root cause:** `resolveElection()` fires 300ms after `handleGatewayAnnounce()` and overrides the Mac win.
**Fix (written, NOT verified):**
- Update `bestElectScore`/`bestElectNodeId` with incoming score on `gateway_announce`.
- Set `electionInProgress = false` when remote score beats own score.
- Written in both `DiscoveryModule.swift` (iOS) and `MeshController.kt` (Android).
**Next action:** test on-device with Android + iPhone + Mac on same WiFi; confirm Mac keeps gateway role through Android VPN toggle.

### Bug #2 — Mac hotspot architectural limit
macOS Internet Sharing cannot share WiFi→WiFi. Needs Ethernet upstream.
**Workaround:** all devices on same WiFi works for local testing.
**Proper fix:** OpenWRT-based router (GL.iNet GL-MT300N-V2), see Chat 10.

### ~~Bug #3 — iOS background VPN~~ — Won't fix (for now)
**Status (2026-04-20):** reclassified from bug to accepted architectural constraint.
`NEPacketTunnelProvider` sandbox blocks `listen()`; `includeAllNetworks` entitlement missing; iOS firewall blocks non-TLS custom protocols tested via NE. After multiple attempts (Chats 06, 06-3), no viable path with current iOS APIs.
**Decision:** iOS is a **foreground-only client**. Same UX model as every commercial VPN on iOS (NordVPN, ExpressVPN, Proton). Users tap Connect → VPN up → traffic routes. Backgrounded → OS suspends after ~3 min. Acceptable MVP behavior.
**Re-evaluate when:** Apple relaxes NE sandbox, OR [C6 Snowflake broker](#c6-snowflake-style-rendezvous-broker-architectural-north-star) lands — WebRTC DataChannel is native to iOS and passes NE/firewall cleanly, making background session continuity more feasible by a different path.

### ~~Bug #4~~ — Traffic relay wiring (code complete 2026-04-20, needs on-device verification)
**Status (2026-04-20):** audit revealed H1 traffic relay was already mostly implemented. Corrected from "not wired" to "wired, unverified."

**Android:** `DiscoveryForegroundService.kt:208-216` already calls `XrayVpnService.restartXrayWithConfig(XrayConfigBuilder.relayConfig(ip, port))` on gateway change, and restores `gatewayConfig()` on becoming gateway. `restartXrayWithConfig` hot-swaps Xray without touching tun2socks or the TUN fd.

**iOS native:** `PacketTunnelProvider.handleAppMessage` accepts `{gateway: ip}`, rebuilds sing-box with SOCKS5 outbound → gateway:10808 via `restartCore(gatewayIP:)`. `VpnModule.setGateway(ip)` persists to app group + notifies NE extension.

**iOS JS glue (added 2026-04-20):** `src/services/VpnService.ts` now subscribes to DiscoveryModule's `gatewayChanged` / `becomeGateway` / `gatewayLost` on iOS and calls `VpnModule.setGateway()` with the mesh gateway IP. `VpnModule.m` ObjC bridge exposes `setGateway:` as a Promise-returning method.

**Needs on-device verification:**
1. Android+iPhone+Mac on same WiFi, Mac connected.
2. On each phone: visit `whatismyip.com` — should show Mac's public IP (or Hetzner's, depending on how Mac is configured).
3. Watch logs: Android `logcat | grep "Remote gateway"`, iOS console `filter restartCore`, Mac console for SOCKS5 inbound connections.
4. Toggle Mac offline → phones should re-elect or fall back to direct VLESS without traffic disruption beyond the election window (~1s).

---

## Critical (blocks beta-to-Iran)

### C1 + C2. Multi-server exit infrastructure — see [EXIT_INFRASTRUCTURE.md](EXIT_INFRASTRUCTURE.md)
Full strategy document written 2026-04-21. Moves from single-Hetzner-VPS to a **dynamic pool of 5-7 servers across 5+ providers in neutral jurisdictions**, discovered via CDN-fronted bootstrap with DoH fallback, rotated on a quarterly + block-triggered schedule.

**Why this is the highest-ROI security investment pre-beta:** closes the single point of failure that a single IP-block attack by Iran could exploit to kill the entire network instantly.

**Phase 1 work (4 weeks):**
- Terraform modules for 2+ hosting providers
- Cloudflare Worker bootstrap service
- Client-side server list fetch + failover
- 3 servers online in 3 different countries

**Phase 2 work (4 weeks):**
- Health monitoring + automated rotation
- Per-server secrets in a secret manager
- DoH fallback path
- Weekly rotation schedule

**Cost:** $40-80/month operational. Break-even at ~40 paying subscribers.

See [EXIT_INFRASTRUCTURE.md](EXIT_INFRASTRUCTURE.md) for: architecture options, server diversity matrix, incident response, comparison to commercial VPNs.

### C3. Per-user VLESS UUID via Telegram bot
Bot issues UUID after TON/USDT payment. Server-side allowlist. Revocable per-user.
**Depends on:** nothing technically; mostly product work.
**Work:** ~1 week including bot.

### C4. Mesh packets encrypted (ChaCha20-Poly1305)
Wrap every UDP datagram. PSK derived from daily SSID/password.
**Depends on:** C5 (so the salt isn't already leaked).
**Work:** ~3 days across all platforms.

### C5. Salt moved server-side
Per-device seed generated on subscription, delivered over VLESS tunnel, stored in Mac keychain / iOS keychain / Android KeyStore.
**Work:** ~2 days.

### C6. Snowflake-style rendezvous broker (architectural north star)
Clients never learn volunteer gateway IPs directly. CDN-fronted broker authenticates users and returns signed WebRTC offers; clients + gateways establish direct DTLS-encrypted P2P. See [GATEWAY_PRIVACY.md](GATEWAY_PRIVACY.md) for full design.
**Why critical:** once volunteer gateways exist (Starlink, expat Macs, volunteer routers), protecting *their* IPs is the dominant attack surface. Every other C-item assumes we solve this.
**Components:**
- Broker on Cloudflare Worker (1 week, depends on C3)
- WebRTC integration on RN client (1 week)
- WebRTC on Mac gateway (4 days)
- Per-user subset assignment (3 days)
- STUN/TURN fallback (2 days)
**Work:** ~3 weeks total. Slots as v2.0 architecture.
**Depends on:** C3 (signed tokens), ideally C1 (bootstrap) running.

---

## High priority

### ~~H1. Wire traffic relay (SOCKS5)~~ — Superseded by H4 on 2026-04-20
H1 wired the mechanism (mesh → hot-swap Xray config → route through gateway). During debugging we discovered the SOCKS5 design leaks destinations to rogue gateways. Pivoted to blind TCP passthrough (H4 below). H1's plumbing (mesh election, config hot-swap, `onGatewayChanged` → `restartXrayWithConfig`) is reused unchanged — only the outbound protocol changes from SOCKS5 to VLESS-to-gateway.

### ~~H4. Blind gateway — TCP passthrough~~ — Attempted 2026-04-20, pivoted
**Result:** implemented end-to-end (Mac `TcpPassthroughForwarder.swift`, Android VLESS outbound to `gateway:4443`, iOS same). Gateway-side bytes flowed correctly (verified via byte counters). **But every session failed with a signature `↑~3.7KB ↓8961B` pattern** — Hetzner's Reality server consistently rejected the phone's ClientHello and served its `www.microsoft.com` masquerade blob (8961B, pixel-consistent across all sessions).

**Root cause undetermined.** Hypotheses:
- Xray client's Reality ClientHello (phone) doesn't match what Hetzner's Reality server (likely sing-box or Xray on X-UI) expects — Reality protocol version skew or uTLS fingerprint mismatch.
- TCP-level characteristics seen by Reality (from Mac's OS) don't match the TLS fingerprint (Chrome-emulated by Xray on Android). Reality may detect this inconsistency and flag as probe.
- Some server-side Reality config detail not visible from the client.

**Status:** code kept in tree (`TcpPassthroughForwarder.swift` exists, forwarder `start()` commented out in `GatewayManager`) but disabled. Reverted to SOCKS5 relay.

**Next step:** H4b below — a pragmatic middle ground that works and still encrypts the phone↔gateway hop.

### H4b. VLESS re-termination gateway (replaces H4)
Gateway runs a VLESS *inbound* on `:4443` (new keys, not Hetzner's) and a VLESS+Reality *outbound* to Hetzner. Phone speaks VLESS to Mac; Mac terminates, re-originates a proven-working VLESS+Reality to Hetzner.

**Tradeoff:** Mac still sees destinations (SOCKS-like semantics from terminated VLESS), but the phone↔Mac hop is encrypted (not raw SOCKS5 over LAN). Fits residual risk A from SECURITY.md — gateway-visibility remains, but at least the LAN traffic is private.

**Implementation:**
- Mac `SingBoxManager`: add a second VLESS inbound on `:4443` with its own UUID + Reality keys (generated on first run, rotated daily).
- Mac's existing outbound to Hetzner remains unchanged.
- Android + iOS `relayConfig`: point VLESS outbound at `gateway_ip:4443` with Mac's keys (delivered via mesh or paired QR).
- Key delivery: put UUID + Reality pubkey in `gateway_announce` message (mesh already encrypted in the v2 plan).

**Work:** ~3 days.
**Depends on:** nothing.
**Limitation:** doesn't achieve the full "blind gateway" property. See [SECURITY.md § Gateway-as-adversary](SECURITY.md#gateway-as-adversary-threat-model) for what this leaves on the table.

### H5. Zero-touch gateway onboarding (phones auto-find & join)
Phone opens OpenBand → taps Connect → system auto-joins the OpenBand SSID using pre-computed credentials (`CredentialGenerator` derives today's SSID+password from shared salt + UTC date).

**Why critical:** matches the real Iran deployment scenario (phones have no direct internet, Mac is the only upstream).

**Status (2026-04-21):**
- ✅ Android `GatewayScannerModule` shipped — calls `WifiNetworkSuggestion` API on API 29+ (Android 10+). User sees a system notification when the SSID is seen in scan; after approval the phone auto-joins forever.
- ✅ Kotlin `CredentialGenerator` bit-exact port of Mac's Swift version — phones compute the same SSID+password that Mac is broadcasting.
- ✅ Wired into `HomeScreen.VPNService.connectAndroid()` — fires on Connect tap.
- ❌ iOS equivalent via `NEHotspotConfigurationManager` — TODO (simpler Apple approval path than Multicast; should be straightforward).
- ❌ Full end-to-end demo — blocked on Mac hotspot needing stable Ethernet / cellular upstream to broadcast `OB-*` for the prompt to trigger.

**Minimum supported platforms:**
- **Android 10+ (API 29)** for auto-join. Android 6–9 returns a graceful "unsupported" result — user must join the OpenBand SSID manually through system WiFi settings. (Android 9 blocks third-party `WifiManager.addNetwork` entirely; pre-Android-10 has no workaround.)
- **iOS 11+** (covered by H5 iOS port).

**Remaining work:**
- iOS `NEHotspotConfigurationManager` (~½ day)
- JS/UX — status screen "Finding gateway..." → "Connected via OB-xxxx" (~½ day)
- Phone-side `GatewayValidator`: after join, try Reality handshake to Hetzner via gateway. If success → stay. If fail → blacklist SSID for 15 min, try next. (~1 day)

**Total remaining work:** ~2 days.
**Depends on:** H4 must land first — H4 reverted to SOCKS5 relay which is already shipped.

### H2. Internet reachability in election score
Periodic probe: `curl --connect-timeout 2 https://<exit-server>:443`.
Multiply existing score by reachability {0, 1}.
**Work:** ~1 day.

### H3. Deploy bootstrap server to Hetzner :3210
Code is written (`bootstrap/server.js`). Needs:
- Open port 3210 in Hetzner firewall
- Run `chmod +x bootstrap/deploy.sh && ./bootstrap/deploy.sh`
- Health: `curl http://<exit-server-ip>:3210/health`
**Work:** ~½ day.

---

## Medium priority

### M1. OpenWRT router package (Chat 10)
Target: GL.iNet GL-MT300N-V2. Runs OpenBand gateway logic + sing-box on MIPS. Solves the "Mac can't share WiFi→WiFi" problem at the hardware layer.
**Work:** ~2 weeks.

### M2. Telegram bot + TON/USDT subscription (Chat 08)
Bot flow: `/start` → show plans → user sends crypto → bot issues VLESS UUID + credentials. Ties into C3.
**Work:** ~1 week.

### M3. Website openband.io on Vercel (Chat 05)
Marketing + download page, FAQ, subscription link to Telegram bot.
**Work:** ~3 days.

### ~~M4. iOS background VPN via WireGuardKit~~ — Won't do (for now)
**Status (2026-04-20):** dropped. See the re-classified "Bug #3" above. iOS ships as foreground-only client. Background VPN is no longer on the roadmap until either Apple changes NE sandbox rules or C6 (WebRTC via broker) provides a different path to session continuity. Chat 06-4 is closed.

---

## Scope decisions — iOS role

**iOS is a client-only platform on OpenBand.** This is a deliberate architectural decision as of 2026-04-20, not a limitation we're still fighting.

| iOS role | Status | Reason |
|----------|--------|--------|
| Foreground VPN client | ✅ Shipped (Chat 04) | Works; sing-box in main process |
| Mesh discovery (foreground) | ✅ Shipped (Chat 02) | `DiscoveryModule.swift` in main app |
| Background VPN | ❌ Won't do | NE sandbox + firewall constraints |
| Background mesh | ❌ Won't do | Same constraints + `listen()` blocked in NE extension |
| Gateway / relay for other users | ❌ Won't do | Requires background + listen sockets |

**Implication:** volunteer gateways run on **Mac** (today) and **OpenWRT routers** (M1). iPhones are always clients. This simplifies the election model (no iOS node ever scores high enough to win) and eliminates a whole class of iOS-specific bugs.

When [C6 Snowflake broker](#c6-snowflake-style-rendezvous-broker-architectural-north-star) ships, iOS clients will connect via WebRTC DataChannel — a path that doesn't hit the NE-sandbox issues at all, since WebRTC is native to iOS and survives backgrounding differently than NEPacketTunnelProvider.

---

## Low priority / nice to have

- Consolidate root-level `HomeScreen/`, `NodesScreen/`, `AccountScreen/` dirs into `src/screens/`.
- Replace default RN README at repo root.
- Remove mis-placed `UdpSocketImpl.kt` from `ios/AirLink/`.
- CI pipeline (GitHub Actions): typecheck, lint, Android build.
- Per-user telemetry (opt-in): is the mesh working? what's the average hop count?

---

## Chat-sprint history

| Chat | Topic | Status |
|------|-------|--------|
| 01 | Full project setup, UI, Android mesh | Done |
| 02 | iOS DiscoveryModule, bidirectional mesh | Done |
| 03 | NEPacketTunnelProvider + Xray arm64 | Done |
| 04 | gomobile bind, HTTP proxy, iOS VPN foreground | Done |
| 05 | Website (openband.io, Vercel) | Pending → M3 |
| 06 | iOS background VPN | **Closed — won't do.** iOS is foreground-only client. |
| 07 | Phase 3: ACK + failover + election | Done |
| 08 | Telegram Bot + TON/USDT subscription | Pending → M2 / C3 |
| 09 | Mac app as router + Bootstrap server | Done (this session) |
| 10 | OpenWRT router package (GL.iNet) | Pending → M1 |

---

## Next session should start with

**H4 — Blind gateway TCP passthrough.** This is the pivot decided 2026-04-20 after discovering the SOCKS5 design let rogue gateways see every user's destinations. H4 is ~2 days of work and is the highest-ROI security improvement possible short-term.

**After H4:** H5 (zero-touch gateway onboarding, ~4 days) — matches the real Iran scenario where phones have no direct internet and must find + join the Mac's WiFi automatically.

**Then:** security cluster C1 / C3 / C5 / H3 — dynamic config, per-user UUIDs, server-side salt, bootstrap deployment.

**Longer-horizon:** C6 (Snowflake-style broker — see [GATEWAY_PRIVACY.md](GATEWAY_PRIVACY.md)) is the v2.0 north star. Don't start it until H4/H5 + C1/C3/C5 land and we have at least 2 external volunteer gateways to test with.
