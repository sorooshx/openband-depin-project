# Project Structure

Last audited: 2026-04-20.
This is a living document — re-audit when adding new top-level directories or native modules.

## Repo roots

- `~/OpenBand/` — React Native mobile app (Android + iOS), bundle `com.openband.app`
- `~/OpenBandMac/` — SwiftUI macOS gateway app (separate Xcode project), bundle `com.openband.mac`

## ~/OpenBand/ (mobile app)

```
OpenBand/
├── App.tsx                     # RN entry, navigation tree
├── index.js                    # RN bootstrap
├── package.json                # RN 0.73.4, Zustand 5, WalletConnect, React Navigation
├── app.json                    # display name: openband
├── babel.config.js
├── metro.config.js
├── tsconfig.json
├── jest.config.js
│
├── HomeScreen/                 # (legacy root-level) HomeScreen.tsx, HomeScreen-old.tsx
├── NodesScreen/                # (legacy root-level) NodesScreen.tsx
├── AccountScreen/              # (legacy root-level, currently empty)
│
├── src/
│   ├── screens/
│   │   └── AccountScreen.tsx
│   ├── components/
│   │   ├── nodes/              # mesh map UI primitives
│   │   └── theme.ts            # colors, spacing, typography tokens
│   ├── services/
│   │   └── VpnService.ts       # VPN + gateway event listeners, JS↔native bridge
│   └── store/
│       ├── useVPNStore.ts      # Zustand store: VPN state, nodes, gateway
│       └── useSimulatedNodes.ts # local mesh simulator for UI dev
│
├── ios/
│   ├── AirLink/                    # main app target
│   │   ├── AppDelegate.{h,mm}
│   │   ├── VpnModule.{swift,m}     # sing-box foreground VPN
│   │   ├── VPNManager.swift        # NE config plumbing
│   │   ├── DiscoveryModule.swift   # mesh discovery + election
│   │   ├── DiscoveryModuleBridge.m
│   │   ├── DiscoveryModuleShared.swift  # neighbor state
│   │   ├── NeighborModule.swift
│   │   ├── NeighborModuleBridge.m
│   │   ├── Info.plist, OpenBand.entitlements
│   │   └── Images.xcassets, LaunchScreen.storyboard
│   ├── PacketTunnel/               # NE extension target (proxy-only)
│   │   ├── PacketTunnelProvider.swift
│   │   ├── ExtensionPlatformInterface.swift
│   │   ├── OBPlatformInterface.{h,m}, PlatformInterface.{h,m}
│   │   └── PacketTunnel.entitlements, PacketTunnel-Bridging-Header.h
│   ├── Frameworks/                 # Mobile.xcframework (sing-box via gomobile)
│   ├── Podfile, Podfile.lock, Pods/
│   ├── OpenBand.xcworkspace        # open THIS, not the xcodeproj
│   └── build/                      # derived data / build artifacts
│
├── android/
│   └── app/src/main/java/com/openband/app/
│       ├── MainActivity.kt, MainApplication.kt
│       ├── XrayVpnService.kt           # VpnService, TUN interface
│       ├── VpnModule.kt, VpnPackage.kt # RN bridge
│       ├── MeshDiscoveryService.kt
│       ├── NativeHelper.kt
│       ├── data/
│       │   ├── MeshController.kt       # ACK, election, keepalive
│       │   ├── MeshProtocol.kt         # JSON wire format
│       │   ├── UdpDiscoveryService.kt  # UDP transport
│       │   ├── UdpSocket.kt, UdpSocketImpl.kt, UdpPacket.kt
│       │   ├── XrayConfigBuilder.kt    # Xray outbound config
│       │   ├── PacketForwarder.kt
│       │   ├── TrafficLogger.kt
│       │   ├── RelayDatabase.kt, RelayDao.kt, RelayEntry.kt
│       │   ├── NeighborRepository.kt, InMemoryNeighborRepository.kt
│       │   └── NodeInfoSerializer.kt
│       ├── domain/
│       │   ├── Neighbor.kt, NodeInfo.kt, NodeMetrics.kt
│       │   ├── NodeRole.kt, NodeRanker.kt, RankedNode.kt
│       └── infrastructure/
│           ├── DiscoveryForegroundService.kt   # foreground service wrapper
│           ├── DiscoveryModule.kt, DiscoveryPackage.kt
│           ├── NeighborModule.kt, NeighborPackage.kt
│           ├── DiscoveryEventBus.kt
│           └── NeighborRepositoryHolder.kt
│
├── bootstrap/                  # Node.js rendezvous server (written, not deployed)
│   └── server.js               # Express on :3210
│
├── docs/                       # ← you are here
├── README.md                   # (default RN readme — replace when onboarding external contributors)
├── TRACKER.md                  # consolidated tracker
└── PROPOSAL.md                 # original proposal
```

## ~/OpenBandMac/ (gateway app)

```
OpenBandMac/
├── OpenBandMac.xcodeproj/
└── OpenBandMac/
    └── Sources/
        ├── Core/
        │   ├── GatewayManager.swift        # orchestrates all subsystems
        │   ├── MeshDiscovery.swift         # UDP :5555, same JSON format as mobile
        │   ├── SingBoxManager.swift        # sing-box subprocess lifecycle
        │   ├── HotspotManager.swift        # Internet Sharing toggle
        │   ├── BootstrapClient.swift       # Hetzner rendezvous client
        │   └── CredentialGenerator.swift   # daily rotating SSID/password
        ├── HomeScreen.swift                # connect + status
        ├── NodesScreen.swift               # live mesh map
        └── AccountScreen.swift             # config, credentials, usage
```

## Shared server infrastructure (not in repo)

- **Exit VPS** `<exit-server-ip>`
  - Xray (VLESS+Reality) on :443, SNI `www.microsoft.com`
  - SOCKS5 on :10808, HTTP proxy on :10809
  - X-UI panel on :2053 path `/<admin-panel-path>/`
  - Bootstrap server planned for :3210 (firewall hole needed)

## Known structural issues

- **Legacy screen dirs at repo root** (`HomeScreen/`, `NodesScreen/`, `AccountScreen/`) duplicate `src/screens/`. Consolidate into `src/screens/` before v1 tag.
- **iOS `UdpSocketImpl.kt`** is mis-placed (Kotlin file in an iOS target). Should be removed or moved.
- **`README.md` at repo root** is still the default RN template. Replace with a real onboarding intro when opening to external contributors.
