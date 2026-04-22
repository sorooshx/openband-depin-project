# Security

**Website:** [https://www.openband.io](https://www.openband.io)

## Threat model

**Adversary:** a state-level actor with
- Deep packet inspection on all transit ISPs
- Ability to block arbitrary IPs, ASNs, SNIs
- Ability to binary-reverse any shipped APK/IPA
- Ability to run active probes against suspicious endpoints
- Control over local WiFi in cafés, hotels, workplaces
- **Ability to operate their own OpenBand gateway nodes** (buy a subscription, run a Mac/router, lure victims)

**Assets we protect:**
1. User's ability to reach the open internet
2. User's identity (which user is using OpenBand right now)
3. User's browsing history (which sites they visit through OpenBand)
4. Server-side credentials (so one user getting caught doesn't kill it for everyone)
5. The mesh graph (who is near whom — could deanonymize users)
6. **IPs of volunteer gateway nodes** (see [GATEWAY_PRIVACY.md](GATEWAY_PRIVACY.md))

**Assets we don't protect (yet):**
- Full metadata resistance (timing, volume analysis) — requires onion routing (C6)
- Perfect forward secrecy against gateway compromise before the [blind gateway pivot](#gateway-as-adversary-threat-model) lands
- User's government ID (not collected)

## Critical issues (must fix before beta)

These are all pre-beta blockers. None are mitigated today.

### 1. Exit server IP hardcoded in binary
**Risk:** Government blocks `<exit-server-ip>` → every OpenBand user loses access simultaneously.
**Fix:** Ship the app with only a bootstrap endpoint (domain-fronted), which hands back server list at runtime. Rotate servers without shipping an update.

### 2. Single shared VLESS UUID
**Risk:** One reverse-engineered APK leaks `<server-uuid>` → gov adds it to their Xray-probe list and the server gets fingerprinted even if IP changes.
**Fix:** Per-user UUIDs issued after subscription (Telegram bot flow in Chat 08). Revocable per-user without touching other users.

### 3. Shared salt in binary
**Risk:** The daily-rotating hotspot SSID/password is `SHA256("<shared-salt>:" + date)`. Anyone with the APK computes tomorrow's SSID, joins the hotspot, sniffs mesh traffic.
**Fix:** Salt is per-router-device, generated server-side at subscription time, written to Mac keychain. Never in binary. Phones fetch it over VLESS tunnel.

### 4. Single exit server
**Risk:** One firewall rule = total network shutdown.
**Fix:** 2–3 exit servers in different countries, different providers. App rotates on failure. Eventually: user picks country.

> **See also:** [GATEWAY_PRIVACY.md](GATEWAY_PRIVACY.md) — once volunteer gateways exist (Starlink, expat operators), protecting *their* IPs is a harder problem than protecting our own exit. Requires a Snowflake-style rendezvous broker and WebRTC P2P so clients never learn gateway IPs directly.

### 5. Mesh packets unencrypted
**Risk:** Anyone on the same WiFi (coffee shop, guest network) captures the whole JSON stream and maps the mesh. Node IDs + relay IPs + neighbor lists all leak.
**Fix:** v2 of [MESH_PROTOCOL.md](MESH_PROTOCOL.md) wraps datagrams in ChaCha20-Poly1305 with a PSK derived from the daily SSID/password.

### 6. No internet quality in election
**Risk:** A node with "internet" that actually can't reach the exit (captive portal, local-only WiFi, DNS blocked) wins election and black-holes all traffic.
**Fix:** Add a "reachability score" to election: periodic `curl --connect-timeout 2` against the exit endpoint. Multiply existing score by reachability {0, 1}.

## Gateway-as-adversary threat model

This section covers the "what if the gateway operator is hostile" threat. It's a separate question from network-level adversaries because the gateway sits on the control path.

### The old design was vulnerable

Prior to 2026-04-20, the gateway was a **SOCKS5 proxy**. To forward traffic, the gateway had to parse each `CONNECT twitter.com:443` request — meaning a rogue operator could log every destination each phone visited. The Reality tunnel only protected the gateway→exit hop, not phone→gateway.

### The fix: WireGuard LAN hop with nested Reality ([ROADMAP H4c](ROADMAP.md#h4c-wireguard-lan-hop-with-nested-reality-end-to-end-replaces-h4b))

Current design (under implementation, 2026-04-22):

- Phone ↔ gateway encrypted with **WireGuard** (ChaCha20-Poly1305, userspace, inside existing sing-box process — no iOS NE sandbox fight).
- **Inside** that WireGuard tunnel, the phone's VLESS+Reality session goes **end-to-end to the exit**.
- Gateway decrypts the outer WireGuard layer; inside it sees only opaque Reality bytes. No destinations, no content, no TLS metadata. Gateway forwards to exit as a blind TCP passthrough.

Earlier design attempts:
- **Blind TCP passthrough (H4)** — tried 2026-04-20, rejected because the exit's Reality server consistently rejected the phone's ClientHello (suspected TCP / uTLS fingerprint mismatch between phone-originated and Mac-forwarded handshakes).
- **VLESS re-termination (H4b)** — scoped but superseded; wouldn't have preserved the full blind-gateway property.

See [ARCHITECTURE.md § Trust model](ARCHITECTURE.md#trust-model) for the full diagram.

### What a rogue gateway can still observe (residual risks)

Even with the blind pivot, a gateway still controls physical transport. These risks remain:

#### Risk A — Timing and volume correlation
Gateway sees the **timing** of each phone's connections and the **byte volume** transferred. Over time, these patterns can fingerprint user behavior even without content:
- "Phone X sent 40 KB request + received 1.2 MB response at 14:32" ≈ a social media page load
- "Phone X kept a connection alive for 45 minutes at 2–4 KB/s" ≈ a video stream
- Correlate with public signals (when does a target go online?) to deanonymize.

**Mitigations:**
- **Traffic padding** — client sends dummy requests at random intervals to blur patterns. Costs bandwidth (~5–10% overhead). Implement in phone's Xray wrapper.
- **Onion routing** ([C6 Snowflake broker](GATEWAY_PRIVACY.md)) — phone uses 2+ gateways, neither knows the (source, destination) pair. Gold-standard fix but ~3 weeks of work.
- **Multi-gateway fanout** — phone splits connections across N gateways. Each gateway sees 1/N of the picture.

#### Risk B — Selective denial / traffic shaping
A gov-run gateway could silently drop specific flows — e.g., refuse to forward connections to `twitter.com`-sized responses. The phone thinks "the internet is slow today" rather than "my gateway is censoring me."

**Mitigations:**
- **Health probes** — phone periodically attempts a connection to a known-good IP through the gateway. If success rate drops below 95%, blacklist gateway, try next.
- **Cross-check at the exit** — exit nodes log successful Reality handshakes. If a user's phone claims "can't connect" but the exit never saw a handshake attempt from their gateway, flag the gateway.
- **Multi-path** — phone keeps a direct VLESS connection to the exit as backup. If gateway-relayed traffic fails, fall back.

#### Risk C — WiFi-level fingerprinting (MAC, probe requests)
Gateway's WiFi access point sees every associated device's MAC address and the list of SSIDs the device probes for. This persists even without any OpenBand traffic.

**Mitigations:**
- **Require MAC randomization** — Android 10+ and iOS 14+ randomize per SSID by default. Add a startup check that warns if randomization is disabled.
- **Random mesh `node_id` per session** — don't persist the same ID across reboots (currently phones use device-stable IDs, which is wrong for this model).
- **Minimize WiFi associations** — phone connects to the gateway SSID only, doesn't probe for saved networks.

#### Risk D — Active probing of the phone
Gateway is on the same local network as the phone. It can send crafted packets trying to fingerprint the phone's OS, detect services, map running apps.

**Mitigations:**
- **Strict firewall on phone** — phone accepts inbound only from the gateway's IP on specific ports (:5555 UDP for mesh ACKs). Drop everything else.
- **Disable AirDrop / Bonjour** when connected to untrusted gateway.
- **No open ports** — phone's Xray binds SOCKS5 inbound to `127.0.0.1` only, not `0.0.0.0`.

#### Risk E — Long-term logging + upload
Gateway can silently log all metadata and upload it to the adversary later (once the user disconnects). Can't be prevented in-device.

**Mitigations:**
- **Assume it happens.** Build everything above as if every gateway uploads logs.
- **Trust no single gateway** — onion routing + multi-gateway fanout distribute the damage.
- **Fresh identity per session** — new mesh `node_id` on every app launch, no persistent client-side cookies in the VPN tunnel.

### Residual-risk summary table

| Risk | Severity today (SOCKS5) | After H4c (WireGuard + nested Reality) | After onion routing (C6) |
|------|-------------------------|----------------------------------------|---------------------------|
| Destinations visible to gateway | 🔴 High | 🟢 None | 🟢 None |
| Traffic content visible | 🟢 None (TLS) | 🟢 None | 🟢 None |
| LAN sniffer sees protocol | 🟡 Medium (SOCKS5 over WPA2) | 🟢 One opaque UDP flow | 🟢 One opaque UDP flow |
| Timing correlation | 🔴 High | 🟡 Medium | 🟢 Low |
| Selective denial | 🟡 Medium | 🟡 Medium | 🟢 Low (N-of-M tolerant) |
| WiFi-level fingerprinting | 🟡 Medium | 🟡 Medium (WiFi layer is separate) | 🟡 Medium |

**H4c is the single highest-ROI security improvement we can make short-term.** It removes destinations from the gateway's view *and* makes the LAN hop look like an opaque UDP flow, which is the dominant pair of threats. Everything else is iterative.

## Mitigated issues

### VLESS+Reality is the exit
TLS-in-TLS with a real SNI — traffic is indistinguishable from a legitimate TLS session to a major HTTPS service. State-level DPI can't distinguish the handshake from a real TLS session. Masquerade target rotates and is not documented publicly. Known good as of 2026-04.

### Multicast leakage via TUN
Early Android builds allowed mesh UDP multicast to go through the VPN tunnel, leaking node IDs upstream. Fixed in `XrayVpnService.kt` by excluding `224.0.0.0/4` and `255.255.255.255` from the VPN routing table.

### ACK storms
Early testing saw phones ACK-flooding the gateway, killing the mesh. Fixed by rate-limiting per message type (see [MESH_PROTOCOL.md](MESH_PROTOCOL.md)).

## Crypto plan (Youtab v2 target)

Layered defense, weakest → strongest:

```
Layer 0: WiFi WPA2/WPA3 on the hotspot (phone↔router)
Layer 1: ChaCha20-Poly1305 on mesh UDP (node↔node, PSK = daily secret)
Layer 2: WireGuard tunnel on the mesh hop (phone↔Mac gateway, post-Phase-4)
Layer 3: VLESS + Reality TLS on exit (Mac/phone↔exit VPS, standard HTTPS port, SNI masquerade)
Layer 4: HTTPS the user's own traffic (app↔destination)
```

Each layer is meaningful even if an outer one is compromised.

## Key rotation

| Secret | Rotation | Method |
|--------|----------|--------|
| Hotspot SSID/password | Daily, UTC midnight | Salt + date SHA256 (pre-beta: salt moves server-side) |
| VLESS UUID | Per-user, on revocation | Telegram bot issues; server-side allowlist |
| Mesh PSK (v2) | Daily, UTC midnight | Derived from SSID/password |
| Server cert for Reality | Per-incident | Manual; cert is published key fingerprint, not TLS cert |

## Incident response playbook

### Server IP appears blocked by a state-level censor
1. Monitor: user reports + exit-node RTT from inside the target region (via friend/VPN).
2. Rotate: spin up a new exit VPS, update bootstrap server's server list.
3. Clients auto-refresh server list on next bootstrap poll (every 10min).
4. Old IP stays alive for 24h as fallback.

### VLESS UUID leaked
1. Revoke at the exit node's admin panel.
2. If widespread: rotate UUID of the leaked subscription tier, force those users to re-register.

### Binary reverse-engineered
Expected. Design assumes this. The secrets in the binary should be minimal (bootstrap domain only).

### Mesh PSK compromise
Next UTC midnight auto-fixes it (new salt = new PSK).

## Security audit status

- ❌ No external audit yet
- ❌ No formal threat model doc in git (this file is the closest thing)
- ❌ No CI-time secret scanning
- ✅ Basic secret hygiene in git (no real UUIDs in public commits — but the hardcoded one IS in source; not worse than shipping it)

## Reporting

Security issues: email `soroushosivand@gmail.com` privately. **Do not** file a public GitHub issue.

## Disclaimer

OpenBand is pre-beta. The security posture is appropriate for **internal testing on trusted WiFi**, not for real use by users in censored regions facing actual state-level threats. Nobody should ship this to production users until every item in [this checklist](#critical-issues-must-fix-before-beta) is ✅.
