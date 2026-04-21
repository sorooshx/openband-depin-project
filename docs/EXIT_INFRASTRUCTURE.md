# Exit Server Infrastructure Strategy

**Status:** architectural direction, not yet implemented.
**Priority:** Critical — blocks beta to Iran. Expands [ROADMAP C1](ROADMAP.md) + [C2](ROADMAP.md) into a full strategy.

---

## 1. Problem statement

OpenBand currently ships with **a single exit server IP hardcoded in the binary**. This is the most concentrated failure mode the system has:

- One block-list addition by an ISP kills OpenBand for every user simultaneously.
- Reality's TLS-masquerade defense is per-server. If the server's public IP is blocked, Reality's elegance becomes irrelevant — the underlying TCP connection never completes.
- One reverse-engineered APK reveals the UUID and public key to blocklists globally.
- One operator mistake (server outage, misconfiguration, legal takedown) = 100% downtime.
- Reality short_id + VLESS UUID are **per-server, per-subscription secrets** that don't scale past ~1 server with the current design.

Single-server is operationally simple but strategically indefensible for a tool targeting a state-level adversary with IP blocking capability.

## 2. The commercial VPN model, and what we borrow from it

Commercial VPN providers (NordVPN, ExpressVPN, Mullvad, Proton) operate **pools of thousands of servers** across many countries, with automated provisioning, rotation, and failover. Their scale is overkill for OpenBand's initial user base, but the *architecture* is directly relevant.

### What they do that we should copy

1. **Server list is never hardcoded in client.** Client contacts a service on launch, receives current list.
2. **Per-server and per-user credentials.** Revoking a server or user doesn't affect others.
3. **Automatic failover.** If one server fails health-check, client rotates to the next.
4. **Geographic diversity.** Users pick a country; providers have servers in 50+.
5. **Infrastructure-as-code.** New server spun up in minutes via Terraform/Ansible.
6. **Burn rate management.** Servers rotated on a schedule + on-demand when blocked.

### What they do that we should NOT copy

1. **Central login service.** Creates a single high-value target — both for blocking and for legal coercion. OpenBand uses decentralized subscription tokens (signed by broker).
2. **Standard VPN protocols** (OpenVPN, IKEv2, WireGuard). Fingerprinted by Iranian DPI in seconds. We need Reality-class masquerading.
3. **Corporate billing.** Leaves paper trail. OpenBand targets crypto payments (TON/USDT) for operational anonymity.
4. **Marketing + public server lists.** Commercial VPNs advertise their IPs publicly. We do the opposite — lists are kept in flux and never published wholesale.

## 3. Architecture options

### Option A — Static list in binary (rejected)

Simply hardcode 5 IPs instead of 1.

**Pros:** trivial to implement.
**Cons:** Same attack class. Reverser adds 5 IPs to blocklist instead of 1. No meaningful gain.

Do not do this.

### Option B — Dynamic fetch via CDN-fronted bootstrap (recommended primary)

App contacts a well-known endpoint on first launch (and periodically after). The endpoint returns the current set of healthy exits along with metadata.

```
Client → HTTPS (domain-fronted) → bootstrap.example.com → returns:
{
  "servers": [
    {"id": "nl-1", "host": "a.b.c.d", "port": 443, "sni": "www.microsoft.com",
     "public_key": "...", "short_id": "...", "country": "NL", "load": 0.3},
    {"id": "de-1", ...},
    ...
  ],
  "ttl_seconds": 600
}
```

Client picks a server based on: latency probe, server load, user preference for country, randomization for load balancing.

**Pros:**
- Rapid rotation — server burns, update the bootstrap list, clients refresh within 10 minutes.
- No app updates needed to recover from blocks.
- Supports per-user policies (subscription tier → server set).
- Small attack surface on the bootstrap endpoint (can run on Cloudflare Workers, ~$0-5/month).

**Cons:**
- Bootstrap endpoint itself becomes a target. Mitigated by:
  - Domain fronting via CDN (Cloudflare / Fastly — Iran has difficulty blocking these without massive collateral damage).
  - Encrypted Client Hello (ECH) where supported.
  - Multi-domain redundancy (3+ bootstrap domains rotated).
- First-run bootstrap requires at least one working domain. If all bootstrap domains are blocked at once, user is offline.

### Option C — DNS-based discovery (supplement)

Use DNS-over-HTTPS (DoH) to resolve a well-known name like `exits.openband.io` to the current IP set. TTL forces periodic refresh.

**Pros:** universal — every OS understands DNS. Low operational cost. DoH bypasses ISP-level DNS hijacking.
**Cons:** Iran regularly performs DNS poisoning. Requires clients to pin DoH resolver (e.g., Cloudflare 1.1.1.1). The poisoning risk is why this is supplement-not-primary.

### Option D — Distributed Hash Table (DHT) — (v3.0 consideration)

Clients participate in a Kademlia-style DHT that gossips the current exit set among themselves. No central authority.

**Pros:** Maximum censorship resistance. No single operator to pressure.
**Cons:** Complex to secure (Sybil attacks, poisoning). Slow bootstrap on fresh install. Bandwidth overhead on clients. Overkill for our current user base. Defer to v3.0.

### Option E — Hybrid with fallback chain (recommended final)

```
1. Try bootstrap over domain-fronted HTTPS (Option B)
2. Failing that, try DoH lookup for fallback domain (Option C)
3. Failing that, use embedded "last resort" list with 2-3 IPs the user
   is told will rotate quarterly via app update
4. Failing that, allow user to manually enter a server config (power users)
```

Each step is an escape hatch. As long as one works, the user gets online.

## 4. Server diversity matrix

Diversity across multiple independent dimensions defeats correlated failure. A single blocklist, subpoena, or outage should not take out multiple servers simultaneously.

| Dimension | Recommendation | Rationale |
|-----------|----------------|-----------|
| **Geographic country** | Min. 5 countries across 3 continents | Avoids single-jurisdiction blocks |
| **Legal jurisdiction** | Favor neutral jurisdictions (NL, DE, IS, PA, SG). Avoid Five Eyes (US/UK/CA/AU/NZ) for Iran target. | Limits legal coercion risk |
| **Hosting provider** | Spread across 5+ providers (Hetzner, OVH, DigitalOcean, Vultr, Contabo, Linode). | Single provider takedown doesn't cascade |
| **ASN** | Servers in different ASNs (not all Hetzner AS24940, etc.) | ISP-level blocks often target ASN ranges |
| **IP range** | Different /24 subnets per server | Subnet-level blocks don't cascade |
| **Payment method** | Crypto preferred; mix of providers that accept crypto; different wallets per server | Avoids billing correlation |
| **Operator** | Some servers volunteer-operated, some project-operated | Distributes operational attack surface |

With 5 servers across this matrix, a targeted takedown of any single dimension loses ≤1 server.

## 5. Operational model

### 5.1 Provisioning

- Infrastructure-as-code via **Terraform modules** per provider (`hetzner.tf`, `ovh.tf`, etc.).
- Configuration management via **Ansible playbook** (`install-xray.yml`) installing Xray + Reality config from templates.
- Per-server secrets generated at provision time: UUID, Reality keypair, short_id. Stored in a **secret manager** (HashiCorp Vault or just encrypted-at-rest git, early stage).
- New server goes live in **≤15 minutes** from terraform apply.

### 5.2 Health monitoring

- **External probes** (preferably from inside Iran via VPN-on-VPS volunteers): every 60s, check each server can complete a VLESS+Reality handshake.
- **Server-side metrics**: CPU, memory, network, active connection count, bandwidth. Push to a central observability endpoint (Grafana / ClickHouse / SQLite-backed).
- **Client-reported failure** telemetry (opt-in): if clients report persistent failures reaching server X, flag it for investigation.

### 5.3 Rotation policy

| Trigger | Action |
|---------|--------|
| Server marked blocked by Iran DPI | Remove from bootstrap list, provision replacement within 4h, terminate old server within 48h |
| Quarterly schedule | Rotate 20% of the pool to prevent long-term fingerprinting |
| Abuse complaints | Rotate within 24h (goodwill with hosting providers) |
| Subscription renewal | Rotate ~monthly regardless (proactive) |

**Burn budget**: plan for 1-2 servers burned per month. Budget replacement cost (~$20-40/mo) accordingly.

### 5.4 Incident response

1. Alert: "server X down" (monitoring).
2. Auto-remove X from bootstrap list.
3. Provision replacement in a different country/provider/ASN.
4. Investigate whether X was blocked (reproduce the block), legal (subpoena), or operational (crashed).
5. If blocked: add X's IP to "burned IPs" log — never reuse. Escalate diversity strategy.
6. If legal: stop serving that jurisdiction.
7. If operational: fix and learn.

## 6. Cost analysis

### Monthly operating costs (5-server baseline)

| Item | Cost |
|------|------|
| 5 × VPS (Hetzner CX11 or equivalent) | $25-50 |
| Bootstrap service (Cloudflare Workers) | $0-5 |
| Monitoring endpoint (small VPS) | $5-10 |
| Domain + DoH | $2-5 |
| Iran-inside probe VPS(s) | $5-10 |
| Terraform / Ansible tooling | $0 (self-hosted) |
| **Total baseline** | **$40-80/month** |

### Per-user economics

- Average user bandwidth: ~5 GB/month conservative (mostly web browsing, some video).
- Server bandwidth limits: ~20 TB/month on a $5 VPS = ~4,000 users per server at 5 GB/month.
- 5 servers = ~20,000 concurrent-user capacity.
- Subscription target: $2/month = revenue $40,000 at full load.
- Break-even: ~40 paying users.

Well within reach for a pre-beta project.

### Scaling economics

Servers are the easy-to-scale cost. Operational complexity (alerts, rotation, incident response) is the binding constraint. One dedicated operator can comfortably manage 20-50 servers; beyond that we need automation/tooling investment.

## 7. Security analysis — what improves, what doesn't

### Improvements

| Attack | Single server | Multi-server | Improvement |
|--------|---------------|--------------|-------------|
| Blocklist single IP | 100% downtime | ~20% capacity loss | Dramatic |
| Compromise one operator's credentials | All users exposed | Only one server's users exposed during rotation window | Material |
| DDoS attack on one IP | All users offline | Other servers still work | Dramatic |
| One hosting provider kicks us off | Total loss | Rebuild in another provider | Material |
| Legal subpoena in one jurisdiction | Full compliance needed | Can migrate the subpoenaed server | Material |
| Reality key leak on one server | All users' next handshakes fail | Only that server affected; other servers unaffected | Material |

### What doesn't improve (by itself)

- **Correlation attacks.** An adversary who can observe traffic patterns across users (or a single user over time) can still fingerprint behavior regardless of which server is the exit.
- **Gateway-as-adversary** (see [GATEWAY_PRIVACY.md](GATEWAY_PRIVACY.md)). Multi-server doesn't change what a hostile gateway can see — that's the gateway-privacy problem, solved separately.
- **Client-side compromise.** If the user's phone is compromised, no server architecture saves them.
- **Protocol fingerprinting.** If DPI can fingerprint Reality handshakes generically (vs VLESS+trojan+shadowsocks), multi-server doesn't help — we need multi-protocol.
- **Iran's backbone-level TLS block.** If the state decides to block all TLS on port 443 that doesn't go to a known-good whitelist, we're offline regardless of how many servers.

### What would need to improve further

To push resilience from "good" to "excellent":

1. **Multi-protocol exits.** Same server runs VLESS+Reality + Shadowsocks-2022 + Trojan + WireGuard, all on different ports with realistic masquerades. Client picks one the ISP isn't blocking today.
2. **Per-region server selection.** Users in Tehran get different servers than users in Mashhad — reduces correlation.
3. **Onion routing** ([GATEWAY_PRIVACY.md](GATEWAY_PRIVACY.md) C6). Multi-hop across uncorrelated operators.
4. **Traffic padding.** Constant-rate dummy traffic to defeat timing analysis.

## 8. Comparison to existing tools

| Tool | Server model | Strengths | Weaknesses |
|------|--------------|-----------|------------|
| Commercial VPN (NordVPN etc.) | 5000+ static, publicly listed servers | Scale, UX | Identified, fingerprintable, not anti-censorship-optimized |
| **Tor** | Thousands of volunteer relays + bridges + meek/snowflake | Strong anonymity, resilient | Low throughput, complex UX |
| **Psiphon** | Dynamic pool, 3 protocols (SSH, TLS-obfuscated) | Anti-censorship-optimized, automatic rotation | Centralized control, closed source history |
| **Lantern** | P2P "giveaway" volunteers + central fallbacks | Peer-based, free tier | Less user trust, complex UI |
| **V2RayN / Shadowsocks resellers** | User-provided server list, manual config | Flexibility | Zero censorship resistance of server list itself |
| **OpenBand (target)** | Dynamic pool via bootstrap + broker + onion | Inherits Reality's masquerade + Snowflake's volunteers | Early-stage, pre-beta |

OpenBand's target positioning is **Psiphon-class censorship resistance with Snowflake-class distributed trust** — neither tool alone covers both dimensions well.

## 9. Implementation phases

### Phase 1 — Minimum viable multi-server (4 weeks)
- Extend ROADMAP C1 + C2 with concrete Terraform modules for 2 providers.
- Bootstrap service on Cloudflare Workers returning a static-for-now JSON list.
- Client code to fetch, pick, and failover.
- 3 servers online in 3 countries.
- **Outcome:** Project no longer dies from a single IP block.

### Phase 2 — Operational maturity (4 weeks)
- Automated health checks + alerting.
- Per-server UUID / Reality keys in a secret manager.
- Client-side bootstrap retry + DoH fallback.
- 5-7 servers across 5+ providers.
- Weekly rotation schedule in place.
- **Outcome:** Project can survive a determined blocking campaign for weeks.

### Phase 3 — Resilience engineering (6 weeks)
- Per-user tokens (ROADMAP C3) so revocation is per-user.
- Iran-inside volunteer probes for real-world health monitoring.
- Multi-protocol exits on each server (Reality + Trojan + WireGuard).
- Geo-aware client selection.
- **Outcome:** Resilient to multi-month sustained blocking pressure.

### Phase 4 — Broker + onion (existing ROADMAP C6)
- Snowflake-style broker.
- Multi-hop onion routing.
- **Outcome:** Production-grade anti-censorship system.

## 10. Open questions

1. **Who operates the bootstrap service?** If it's a single operator, that's a single legal / political attack vector. Consider multi-operator bootstrap with independent instances.
2. **How do we fund beyond 50 users?** Subscription at $2/month buys 5 GB outbound per user — needs to cover bandwidth overage, scaling infra, operator time. TON/USDT flow is OK, but operator taxation / AML compliance in some jurisdictions is non-trivial.
3. **Can we decentralize exit operation?** Volunteer-run exits require trust (they see encrypted Reality bytes, but they know *who* their clients are). This is a different problem from volunteer gateways (who see destinations). Worth exploring, but non-trivial.
4. **What's the target user count at v1.0?** Our current infrastructure scales to ~20k users. Above that, we need proper observability, incident-response automation, and likely dedicated DevOps.
5. **Legal structure?** Running even 5 servers as a project raises questions about liability, especially if users do anything illegal through them. Consider a non-profit structure (similar to Tor Project Inc.) or a jurisdiction-neutral legal entity.

## 11. Recommendation summary

Move from single Hetzner VPS to a **dynamic pool of 5-7 servers across 5+ providers in neutral jurisdictions, discovered via domain-fronted bootstrap service with DoH fallback, rotated on a quarterly + block-triggered schedule**. Budget ~$40-80/month initial operating cost, break-even at ~40 paying subscribers.

This is **the single most valuable security investment OpenBand can make pre-beta**. It closes the largest current vulnerability (single point of failure) and unlocks the path to onion routing and zero-touch onboarding at scale.

Integrate into ROADMAP: replace C1 + C2 with this document's Phase 1 + 2, keep C6 (broker) and GATEWAY_PRIVACY.md (onion) as the v2.0 north star.
