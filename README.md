# OpenBand

**A decentralized physical-infrastructure network (DePIN) for censorship-resistant internet — combining local WiFi mesh routing with satellite-capable gateway nodes.**

[![Status: Pre-beta](https://img.shields.io/badge/status-pre--beta-orange)](docs/ROADMAP.md)
[![Architecture: DePIN](https://img.shields.io/badge/architecture-DePIN-purple)](#the-three-pillars)
[![Docs: Public](https://img.shields.io/badge/docs-public-brightgreen)](docs/)
[![Whitepaper](https://img.shields.io/badge/whitepaper-read-blue)](WHITEPAPER.md)
[![License: TBD](https://img.shields.io/badge/license-TBD-lightgrey)](LICENSE)

**Website:** [https://www.openband.io](https://www.openband.io)

OpenBand is a censorship-circumvention system for users in jurisdictions with state-level traffic filtering. It solves the problem that **centralized VPNs have a single point of failure** — one IP block kills the service — by doing three things at once:

1. **DePIN** — gateways are run by a distributed network of volunteer operators (laptops, OpenWRT routers, satellite-connected devices), incentivized rather than centrally owned. No single operator, company, or jurisdiction controls the network.
2. **Mesh networking** — phones inside the censored region discover nearby gateways over local WiFi and auto-elect the best one. No pre-shared secrets, no manual configuration, no central directory.
3. **Satellite-capable uplink** — gateways can run on any upstream including **satellite (Starlink, Iridium, etc.)**, completely bypassing terrestrial ISPs. A state cannot block orbital traffic at its borders.

The end user opens an app, it joins the nearest OpenBand gateway automatically, and traffic flows through an encrypted end-to-end tunnel (VLESS+Reality, masqueraded as ordinary HTTPS) to a volunteer-operated exit server abroad.

## The three pillars

### 🌐 DePIN — distributed operator infrastructure
Gateway operators are not employees of a VPN company. They are volunteers (and eventually subscription-compensated participants) running OpenBand gateway software on commodity hardware — a Mac laptop in an expat home, a GL.iNet router on a café table, a Starlink terminal in a vehicle. The network's resilience scales with the number of independent operators, not with a company's server fleet. This is the Tor / Snowflake / Helium model adapted for anti-censorship.

### 📡 Mesh networking — zero-config local discovery
Inside the censored region, phones running OpenBand broadcast and listen on UDP port 5555. They discover each other, elect a gateway by score (internet reachability, signal strength, battery), and tunnel traffic through the winner. A Mac or router nearby scores highest and always wins — phones never need to know gateway IPs in advance. The wire format is documented in [MESH_PROTOCOL.md](docs/MESH_PROTOCOL.md).

### 🛰️ Satellite and alternative uplinks — censorship-proof backhaul
Because gateways are ordinary devices running in operator environments outside the censored region, they can route their upstream through anything: fiber, cable, 5G, **Starlink**, Iridium, or other LEO/MEO satellite services. The adversary controls its national border, not low-Earth orbit. A gateway in a country with repressive ISPs can still reach a working exit via satellite. This is especially relevant for operators inside *partially-censored* regions who want to help users in more censored regions.

Together, these three make OpenBand a **decentralized, self-organizing, backhaul-agnostic** alternative to the VPN-with-static-server model that's been failing censorship tests for years.

## How it's funded

- **Today (pre-launch):** 100% donation-funded. Donations denominated in **USDC on Base** (Coinbase's Ethereum L2). Donations directly pay gateway-operator rewards, Starlink/satellite reimbursements for operators running satellite-backed nodes, and exit-server hosting costs.
- **After launch:** subscriptions at ~USD 2/month, paid in USDC on Base — no corporate billing, no KYC, on-chain anonymous payments. No project-specific token, no ICO, no governance coin.
- **🌍 Free during blackouts:** when Iran (or any target region) experiences a state-level internet blackout or major circumvention-tool ban, OpenBand is free for users in that region for the duration plus a cooldown period. Detected automatically via OONI, IODA, Cloudflare Radar, and operator reports. No price gate when people need this most.
- **No ads. No data sales. No "partnerships."** Revenue exists solely to keep the network alive.

---

## Start here

- 📄 **[Whitepaper](WHITEPAPER.md)** — the single-document overview: problem, threat model, design, and honest status
- 🏗️ **[Architecture](docs/ARCHITECTURE.md)** — how the pieces fit together
- 🛡️ **[Security Analysis](docs/SECURITY.md)** — threat model, what we protect, what we don't
- 🛣️ **[Roadmap](docs/ROADMAP.md)** — what's shipped, what's next, pre-beta blockers

## The one-paragraph pitch

Phones inside a censored region run a small mobile app. The app discovers nearby OpenBand gateways over local WiFi **mesh**, elects the strongest one automatically, and tunnels traffic through it to an exit server outside the censored region. Gateway operators are a distributed network of **DePIN** volunteers running OpenBand software on commodity hardware — Macs, OpenWRT routers, **Starlink-connected nodes**. The exit protocol (VLESS+Reality) masquerades as a real HTTPS connection to a high-trust domain, so DPI can't distinguish it from normal web traffic. Because gateways can run on **satellite uplinks** bypassing national ISPs entirely, and because there are many small gateway operators rather than a handful of big servers, a state adversary cannot block the network with a single firewall rule — they'd have to block satellite orbits or enumerate every residential IP in the world.

## Why this exists

Iran and similar regimes have made traditional VPN tools increasingly ineffective:

- Commercial VPN IPs are enumerated and blocked within days.
- Protocol fingerprinting (OpenVPN, IKEv2, WireGuard on known ports) is instant.
- Tor bridges get probed actively.
- Single points of failure — a single firewall rule kills most services.

OpenBand's approach, inspired by [Tor's Snowflake](https://snowflake.torproject.org/), distributes the exit surface across many small volunteer-operated nodes. One burned gateway affects ~1/N of users, where N can be hundreds or thousands. This is the same strategy that has kept Snowflake viable in hostile environments for years.

---

## Repository contents

This is a **public documentation and whitepaper repository**. The source code of the OpenBand mobile apps, gateway software, and exit server configuration lives in a separate private repository (for operational security — see the [whitepaper](WHITEPAPER.md) and [security doc](docs/SECURITY.md) for why).

```
openband-depin-project/
├── README.md                          ← you are here
├── WHITEPAPER.md                      ← single-doc system overview
├── LICENSE                            ← TBD (CC-BY-4.0 for docs, AGPL-3.0 for code)
└── docs/
    ├── ARCHITECTURE.md                ← system design + diagrams
    ├── SECURITY.md                    ← threat model + residual risks
    ├── MESH_PROTOCOL.md               ← on-wire protocol specification
    ├── GATEWAY_PRIVACY.md             ← broker + onion-routing design
    ├── EXIT_INFRASTRUCTURE.md         ← multi-server exit strategy
    ├── ROADMAP.md                     ← engineering plan
    ├── CHANGELOG.md                   ← version history
    ├── PROJECT_STRUCTURE.md           ← repo layout (of the private code repo)
    └── CONTRIBUTING.md                ← how to contribute
```

All real credentials (server IPs, UUIDs, cryptographic keys, salt values) have been replaced with placeholders like `<exit-server-ip>` and `<shared-salt>`. Operational values are managed separately via operator-only secret storage.

---

## Current status (pre-beta)

**What works today:**
- Mesh discovery across Android, iOS, and macOS
- Automatic gateway election (Mac wins predictably over mobile nodes)
- SOCKS5 relay through gateway → VLESS+Reality → exit server → internet
- Zero-touch WiFi onboarding on Android 10+ and iOS 11+
- Full exit traffic verified end-to-end in cooperative test conditions

**Pre-beta blockers** (we will not ship to real at-risk users until these close):
1. Exit server IP hardcoded in binary (needs dynamic bootstrap — see [EXIT_INFRASTRUCTURE.md](docs/EXIT_INFRASTRUCTURE.md))
2. Shared VLESS UUID (needs per-user issuance)
3. Shared salt compiled into binary (needs server-side delivery)
4. Single exit server (needs 5–7 redundant)
5. Unencrypted mesh packets
6. No internet-reachability check in gateway election
7. Gateway sees destinations (needs VLESS re-termination)

Full discussion: [SECURITY.md](docs/SECURITY.md).

---

## How to contribute

We welcome the following kinds of contributions:

- **Adversarial review.** Find flaws in the design. Report security issues privately first; see below.
- **Protocol review.** Read the [whitepaper](WHITEPAPER.md) and [MESH_PROTOCOL.md](docs/MESH_PROTOCOL.md). Challenge assumptions.
- **Running a gateway.** When gateway software reaches volunteer-operator stage (target: early beta), running one is the highest-leverage contribution to network resilience.
- **Operational help.** Sysadmin / SRE expertise for exit server rotation, monitoring, and failover is always valuable.
- **Documentation and translation.** The whitepaper and user-facing docs would benefit from translation into Farsi, Arabic, and other high-priority languages.

See [CONTRIBUTING.md](docs/CONTRIBUTING.md) for workflow.

## Security reporting

**Do not file public issues for security vulnerabilities.** Use one of:

- GitHub Security Advisory (preferred): [Create private advisory](https://github.com/SorooshTechnologies/openband-depin-project/security/advisories/new)
- Email: (project owner's contact — to be added)

We request a 30-day disclosure window for critical issues; longer negotiable for hard-to-fix vulnerabilities.

## Acknowledgements

OpenBand stands on the shoulders of decades of anti-censorship research:

- **The Tor Project** — Snowflake, meek, bridges, and the entire threat-model playbook
- **Xray / sing-box contributors** — VLESS and Reality protocols
- **Signal** — cryptographic protocol design standards
- **WireGuard** — modern minimal encrypted transport
- **Psiphon, Lantern, Briar, Bramble** — tools and prior art for mesh + multi-protocol circumvention

Mistakes are ours; foundations are theirs.

## License

Documentation: **CC-BY-4.0** (free to share and adapt with attribution).
Source code (private repo): **AGPL-3.0** (target).

See [LICENSE](LICENSE) for final terms.

---

*This is a pre-beta research and development project. It is not yet safe for production use by real at-risk users in hostile jurisdictions. The roadmap tracks the path to a defensible beta release. Watch the repository for updates.*
