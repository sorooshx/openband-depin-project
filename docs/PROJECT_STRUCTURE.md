# Project Structure

**Website:** [https://www.openband.io](https://www.openband.io)

Last audited: 2026-04-21.

High-level layout of the OpenBand codebase. Operational detail (exit-server addresses, SNI masquerade targets, port assignments, admin paths, credential-derivation internals, application bundle identifiers) is intentionally omitted from the public copy. Contributors with commit access get a fuller version in the private repository.

## Repo roots

- **Mobile app** — React Native (Android + iOS)
- **Gateway app** — SwiftUI macOS app (separate Xcode project)
- **Bootstrap / broker** — Node.js (pre-launch rendezvous)

## Mobile app

```
OpenBand/
├── App.tsx                     # RN entry, navigation tree
├── index.js                    # RN bootstrap
├── package.json                # RN 0.73.4, Zustand 5, React Navigation
├── babel.config.js
├── metro.config.js
├── tsconfig.json
├── jest.config.js
│
├── src/
│   ├── screens/                # HomeScreen, NodesScreen, AccountScreen
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
│   ├── App/                        # main app target
│   │   ├── AppDelegate.{h,mm}
│   │   ├── VpnModule.{swift,m}     # foreground VPN bridge
│   │   ├── VPNManager.swift        # NetworkExtension config plumbing
│   │   ├── DiscoveryModule.swift   # mesh discovery + election
│   │   ├── DiscoveryModuleBridge.m
│   │   ├── DiscoveryModuleShared.swift
│   │   ├── NeighborModule.swift
│   │   └── NeighborModuleBridge.m
│   ├── PacketTunnel/               # NetworkExtension target (proxy-only)
│   │   ├── PacketTunnelProvider.swift
│   │   ├── ExtensionPlatformInterface.swift
│   │   └── OBPlatformInterface.{h,m}, PlatformInterface.{h,m}
│   └── Frameworks/                 # embedded proxy framework (built via gomobile)
│
├── android/
│   └── app/src/main/java/…/
│       ├── MainActivity.kt, MainApplication.kt
│       ├── XrayVpnService.kt             # VpnService, TUN interface
│       ├── VpnModule.kt, VpnPackage.kt   # RN bridge
│       ├── MeshDiscoveryService.kt
│       ├── NativeHelper.kt
│       ├── data/
│       │   ├── MeshController.kt         # ACK, election, keepalive
│       │   ├── MeshProtocol.kt           # JSON wire format
│       │   ├── UdpDiscoveryService.kt    # UDP transport
│       │   ├── UdpSocket*.kt, UdpPacket.kt
│       │   ├── XrayConfigBuilder.kt      # proxy outbound config
│       │   ├── PacketForwarder.kt
│       │   ├── TrafficLogger.kt
│       │   ├── RelayDatabase.kt, RelayDao.kt, RelayEntry.kt
│       │   ├── NeighborRepository.kt, InMemoryNeighborRepository.kt
│       │   └── NodeInfoSerializer.kt
│       ├── domain/
│       │   ├── Neighbor.kt, NodeInfo.kt, NodeMetrics.kt
│       │   └── NodeRole.kt, NodeRanker.kt, RankedNode.kt
│       └── infrastructure/
│           ├── DiscoveryForegroundService.kt
│           ├── DiscoveryModule.kt, DiscoveryPackage.kt
│           ├── NeighborModule.kt, NeighborPackage.kt
│           ├── DiscoveryEventBus.kt
│           └── NeighborRepositoryHolder.kt
│
├── bootstrap/                  # Node.js rendezvous server (pre-launch)
│   └── server.js
│
└── docs/                       # ← you are here
```

## Gateway app (macOS)

```
OpenBandMac/
└── Sources/
    ├── Core/
    │   ├── GatewayManager.swift        # orchestrates all subsystems
    │   ├── MeshDiscovery.swift         # same JSON wire format as mobile
    │   ├── SingBoxManager.swift        # proxy subprocess lifecycle
    │   ├── HotspotManager.swift        # Internet Sharing toggle
    │   ├── BootstrapClient.swift       # rendezvous client
    │   └── CredentialGenerator.swift   # rotating SSID / password
    ├── HomeScreen.swift                # connect + status
    ├── NodesScreen.swift               # live mesh map
    └── AccountScreen.swift             # config, credentials, usage
```

## Shared server infrastructure

Not included in this repository. Exit nodes run standard VLESS + Reality with rotating parameters. Operational configuration is managed out-of-band.

## Target platforms

| Role | Platform | Status |
|---|---|---|
| Client | Android 8+ | shipping |
| Client | iOS 14+ (foreground) | shipping |
| Gateway | macOS 13+ | shipping |
| Gateway | OpenWRT / Linux | roadmap |
| Exit | Linux VPS (any jurisdiction) | shipping |

## Related reading

- [ARCHITECTURE.md](ARCHITECTURE.md) — system diagram and trust model
- [MESH_PROTOCOL.md](MESH_PROTOCOL.md) — LAN discovery wire format
- [GATEWAY_PRIVACY.md](GATEWAY_PRIVACY.md) — rendezvous-broker design (v2.0 target)
- [SECURITY.md](SECURITY.md) — threat model and residual risks
