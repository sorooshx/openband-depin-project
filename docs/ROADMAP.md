# Roadmap

**Website:** [https://www.openband.io](https://www.openband.io)

Status as of **2026-04-22**.
Priority is strict: do the Critical items before anything in High, etc.

---

## Current sprint — security hardening (2026-04-22 → ~2 weeks)

Focused push that closes every unfixed LAN-side leak before moving on to multi-server exit:

1. **H4c** — WireGuard LAN hop + nested Reality (~12d, see below)
2. **Per-session `node_id`** — closes Residual Risk C (~1d)
3. **MAC-randomization startup check** — warns if OS-level randomization is disabled (~1d)
4. **Disable mDNS / AirDrop on VPN start** — closes RF-chatter leak (~1d)
5. **H3** — deploy bootstrap server (~½d)

After this sprint, the gateway-as-adversary threat model is essentially closed short of onion routing. Next sprint: **C1/C2 multi-server exit**, the real pre-beta blocker.

---

## What works today (confirmed)

- ✅ Android VPN tunnel to exit node via VLESS+Reality
- ✅ iOS VPN foreground mode (sing-box in main app process)
- ✅ Mesh discovery on all 3 platforms (Android, iOS, Mac)
- ✅ Gateway election — Mac wins with score 1.0 in steady state
- ✅ `gatewayChanged` / `becomeGateway` / `gatewayLost` JS events
- ✅ Mac sing-box tunnel confirmed (exit VPS IP verified via egress IP check)
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

**iOS native:** `PacketTunnelProvider.handleAppMessage` accepts `{gateway: ip}`, rebuilds sing-box with SOCKS5 outbound to the gateway via `restartCore(gatewayIP:)`. `VpnModule.setGateway(ip)` persists to app group + notifies NE extension.

**iOS JS glue (added 2026-04-20):** `src/services/VpnService.ts` now subscribes to DiscoveryModule's `gatewayChanged` / `becomeGateway` / `gatewayLost` on iOS and calls `VpnModule.setGateway()` with the mesh gateway IP. `VpnModule.m` ObjC bridge exposes `setGateway:` as a Promise-returning method.

**Needs on-device verification:**
1. Android+iPhone+Mac on same WiFi, Mac connected.
2. On each phone: visit `whatismyip.com` — should show Mac's public IP (or the exit node's, depending on how Mac is configured).
3. Watch logs: Android `logcat | grep "Remote gateway"`, iOS console `filter restartCore`, Mac console for SOCKS5 inbound connections.
4. Toggle Mac offline → phones should re-elect or fall back to direct VLESS without traffic disruption beyond the election window (~1s).

---

## Critical (blocks beta deployment)

### C1 + C2. Multi-server exit infrastructure — see [EXIT_INFRASTRUCTURE.md](EXIT_INFRASTRUCTURE.md)
Full strategy document written 2026-04-21. Moves from a single exit VPS to a **dynamic pool of 5-7 servers across 5+ providers in neutral jurisdictions**, discovered via CDN-fronted bootstrap with DoH fallback, rotated on a quarterly + block-triggered schedule.

**Why this is the highest-ROI security investment pre-beta:** closes the single point of failure that a single IP-block attack by a state-level censor could exploit to kill the entire network instantly.

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

### C3. Per-user VLESS UUID via Telegram bot + USDC/Base subscription
Bot issues UUID after **USDC-on-Base** subscription payment. Server-side allowlist. Revocable per-user. Free-during-blackout: bot auto-issues without payment when OONI/IODA/Cloudflare Radar confirm a state-level outage in the user's region (see WHITEPAPER § 11.3). Gateway-operator rewards + Starlink reimbursements paid from donation pool today, from subscription pool post-launch.
**Depends on:** nothing technically; mostly product + on-chain smart contract work.
**Work:** ~1 week bot + 1 week Base smart contract + frontend = ~2 weeks total.

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
**Result:** implemented end-to-end (Mac `TcpPassthroughForwarder.swift`, Android VLESS outbound to a gateway port, iOS same). Gateway-side bytes flowed correctly (verified via byte counters). **But every session failed with a distinctive upstream-bytes / downstream-bytes signature** — the exit node's Reality server consistently rejected the phone's ClientHello and served its SNI masquerade blob (pixel-consistent across all sessions).

**Root cause undetermined.** Hypotheses:
- Xray client's Reality ClientHello (phone) doesn't match what the exit's Reality server expects — Reality protocol version skew or uTLS fingerprint mismatch.
- TCP-level characteristics seen by Reality (from Mac's OS) don't match the TLS fingerprint (Chrome-emulated by Xray on Android). Reality may detect this inconsistency and flag as probe.
- Some server-side Reality config detail not visible from the client.

**Status:** code kept in tree (`TcpPassthroughForwarder.swift` exists, forwarder `start()` commented out in `GatewayManager`) but disabled. Reverted to SOCKS5 relay.

**Next step:** H4b below — a pragmatic middle ground that works and still encrypts the phone↔gateway hop.

### ~~H4b. VLESS re-termination gateway~~ — Superseded by H4c on 2026-04-22
Scoping considered a second VLESS inbound on the gateway with its own Reality keys. Abandoned in favor of H4c (WireGuard + nested Reality), which is cleaner, gives better iOS ergonomics, and preserves the full blind-gateway property that VLESS re-termination could not.

### H4c. WireGuard LAN hop with nested Reality end-to-end (replaces H4b)
Encrypt the phone ↔ gateway LAN hop with **WireGuard** (userspace, inside the Xray/sing-box process). Inside that tunnel, the phone's VLESS+Reality session goes **end-to-end to the exit node** unchanged. The gateway decrypts WireGuard, sees only opaque Reality bytes, and forwards them to the exit — no destinations, no content, no TLS metadata.

```
[ WG[ Reality[ app ] ] ]   UDP :51820           [ Reality[ app ] ]     TCP :443
Phone ─────────────────────────────────→  Gateway ────────────────────────→  Exit
                                           (WG term + blind forward)
```

**Why WireGuard and not VLESS on the LAN leg:**
- Modern primitives (ChaCha20-Poly1305, Curve25519), ~20% less phone CPU than TLS
- UDP on a fixed LAN port — no DPI concerns inside our own WiFi
- Clean iOS story: run WG in **userspace** inside the existing sing-box process, no NE whole-device sandbox fight (the constraint that blocked iOS background VPN doesn't apply here)
- Official libs: `wireguard-android` (AOSP) and `WireGuardKit` (Swift Package)

**Gateway target: OpenWRT first, not macOS** (decided 2026-04-22).
On macOS, userspace `wireguard-go` requires root to create `utun` devices (SYSPROTO_CONTROL is privileged). A production-quality Mac gateway would require an SMAppService privileged helper with XPC — ~3-4 days extra, and the Mac gateway is a dev-testing convenience, not the production target. On OpenWRT, kernel WireGuard is native and privileged already. The real production deployment scenario (Starlink → OpenWRT router → phones on WiFi) also runs on OpenWRT.

So H4c collapses partially into M1: the gateway-side work happens directly on OpenWRT. Mac keeps running legacy SOCKS5 for local dev. Phone-side code is identical either way.

**Work breakdown (revised ~9 days total):**
| Task | Est. | Status |
|------|------|--------|
| Mesh protocol extension — `gateway_announce` carries `wg_pubkey` + `wg_port` | 1d | ✅ shipped 2026-04-22 |
| `peer_register` mesh control message — phones publish their WG pubkey to gateway | 0.5d | ✅ shipped 2026-04-22 |
| Android Xray config — WireGuard outbound chained through Reality | 2d | ✅ shipped 2026-04-22 |
| iOS sing-box config — WireGuard outbound chained through Reality | 2d | ✅ shipped 2026-04-22 |
| Linux/OpenWRT gateway daemon scaffold (Python reference implementation) | 1d | ✅ scaffolded 2026-04-22; restart-safe + SIGTERM-clean |
| End-to-end test against the Linux daemon | 1d | 🚧 pending — needs Linux VM or RPi on phone's WiFi |
| OpenWRT `.ipk` package (`procd` service, `nftables` instead of `iptables`, UCI integration) | 2d | 🚧 pending — needs GL.iNet hardware |
| iOS battery profile under sustained WG | 0.5d | pending |

**Mac gateway (deferred):** stays on SOCKS5 as the dev-testing path. If we ever want WG on Mac for production, we'll add an SMAppService privileged helper as a separate sprint.

**Phone-side library choice:** Xray and sing-box both ship a **WireGuard outbound** protocol natively. No separate `wireguard-android` / `WireGuardKit` integration needed — the phone's existing proxy engine adds WG as a transport beneath its existing Reality outbound. Cuts the integration effort roughly in half.

**Open questions:**
- **Peer auth:** ship **open** (any phone can peer) pre-C3; tighten to token-gated (C3 UUID as WG PSK) when the Telegram bot lands.
- **Keypair lifetime:** per-phone with monthly rotation — simpler UX than per-session, still supports revocation.

**Depends on:** for the OpenWRT terminator, GL.iNet hardware or a Linux VM as a test gateway.
**Unlocks:** closes Residual Risk A (destinations visible to gateway) for the LAN hop. Matches the real production deployment scenario exactly.

### H5. Zero-touch gateway onboarding (phones auto-find & join)
Phone opens OpenBand → taps Connect → system auto-joins the OpenBand SSID using pre-computed credentials (`CredentialGenerator` derives today's SSID+password from shared salt + UTC date).

**Why critical:** matches the real production deployment scenario (phones have no direct internet, Mac is the only upstream).

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
- Phone-side `GatewayValidator`: after join, try Reality handshake to the exit via gateway. If success → stay. If fail → blacklist SSID for 15 min, try next. (~1 day)

**Total remaining work:** ~2 days.
**Depends on:** H4 must land first — H4 reverted to SOCKS5 relay which is already shipped.

### H2. Internet reachability in election score
Periodic probe against the exit endpoint with a short connect timeout.
Multiply existing score by reachability {0, 1}.
**Work:** ~1 day.

### H3. Deploy bootstrap server
Code is written (`bootstrap/server.js`). Deployment steps (firewall rules, service install, health check) are operational detail, not published.
**Work:** ~½ day.

---

## Medium priority

### M1. OpenWRT router package (Chat 10)
Target: GL.iNet GL-MT300N-V2 or GL-AR750S. The Python reference daemon already implements the full H4c gateway-side logic (WG inbound, dynamic peer add via `peer_register`, `gateway_announce` broadcast). M1 is mostly **packaging**, not new protocol work:

- Wrap as a `procd` service (`/etc/init.d/openband-gateway`)
- Replace `iptables` with `nft` / OpenWRT firewall UCI
- Build into the stock firmware image / OpenWRT feed
- Hardware validation: thermals, throughput, MIPS-arch quirks
- WiFi AP config so phones associate to the router's SSID directly

**Work:** ~1 week, contingent on GL.iNet hardware on hand.

### M1b. Multi-router mesh routing (router-only fabric)
Once two+ OpenWRT gateways exist in the same physical area, run a routing protocol over inter-router WireGuard tunnels so packets find the best path to internet automatically. Phones don't change — they pick the nearest router; the routing protocol takes care of the rest.

- Engine choice: **`babel`** (lightweight, L3, works directly over WireGuard) preferred over `batman-adv` (kernel module, L2). `babeld` is a single binary with a small config.
- Each router maintains WG tunnels to neighbor routers it discovers via the existing mesh announce.
- Routers with internet upstream advertise a default route; routers without learn the best path.
- Failover is automatic — when the upstream router dies, `babel` re-converges within ~10s.
- Election logic in OpenBand becomes a **fallback for phone-only cells** (no router present); under multi-router mesh it doesn't run.

**Work:** ~1–2 weeks on top of M1. Doesn't block M1 shipping — single-router deployments work standalone.

### M2. Telegram bot + USDC-on-Base subscription (Chat 08)
Bot flow: `/start` → show plans → user pays USDC on Base → on-chain payment confirmed → bot issues VLESS UUID + credentials. Ties into C3. Includes blackout-detection auto-waiver logic.
**Work:** ~2 weeks (bot + Base smart contract + blackout-signal integration).

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

**Implication:** the production gateway is the **OpenWRT router** (M1). Mac is dev-only. iPhones are always clients. This simplifies the election model (no iOS node ever scores high enough to win), eliminates a whole class of iOS-specific bugs, and keeps the volunteer-operator pitch coherent — "host a router and earn rewards" is unambiguous.

When [C6 Snowflake broker](#c6-snowflake-style-rendezvous-broker-architectural-north-star) ships, iOS clients will connect via WebRTC DataChannel — a *client-only* path that doesn't hit the NE-sandbox issues at all, since WebRTC is native to iOS and survives backgrounding differently than NEPacketTunnelProvider. C6 does not change iOS from client-only.

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
| 08 | Telegram Bot + USDC-on-Base subscription | Pending → M2 / C3 |
| 09 | Mac app as router + Bootstrap server | Done (this session) |
| 10 | OpenWRT router package (GL.iNet) | Pending → M1 |

---

## Next session should start with

**End-to-end H4c validation against the Linux daemon.** Phone-side WG outbound is shipped on both platforms; the gateway daemon (Python reference implementation) is scaffolded and restart-safe. What's missing is real-traffic validation:

1. Run the daemon on a Linux box (RPi, Lima VM with bridged networking, or any spare Linux host) on the same WiFi as a test phone.
2. Verify `wg show wg0` shows the phone's pubkey after Connect.
3. Confirm `tcpdump -i wg0` shows only encrypted Reality bytes — no plaintext destinations (the blind-gateway property).
4. Confirm web traffic exits via the gateway, then the exit server, then to the destination.

**After validation:** M1 — package the daemon as an OpenWRT `.ipk` (procd service, nftables instead of iptables, UCI integration) once GL.iNet hardware is on hand. Roughly 1 week.

**Parallel track:** security cluster C1 / C3 / C5 / H3 — dynamic config, per-user UUIDs, server-side salt, bootstrap deployment. Independent of the H4c validation work.

**Longer-horizon:** C6 (Snowflake-style broker — see [GATEWAY_PRIVACY.md](GATEWAY_PRIVACY.md)) is the v2.0 north star. Don't start it until H4/H5 + C1/C3/C5 land and we have at least 2 external volunteer gateways to test with.
