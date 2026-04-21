# Mesh Protocol

Wire-level spec for node discovery, gateway election, and keepalive.
Current version: **v1** (unencrypted JSON over UDP). v2 will wrap this in ChaCha20-Poly1305.

## Transport

- **Protocol:** UDP
- **Port:** `5555`
- **Encoding:** UTF-8 JSON, no framing prefix, one message per datagram
- **MTU:** keep payloads under 1200 bytes
- **Multicast:** Android broadcasts to `255.255.255.255:5555`; iOS uses `Network.framework` unicast to known peers + broadcast on announce; Mac does both

## Message envelope

Every message has these fields:

```json
{
  "type": "heartbeat",
  "node_id": "Mobile_A914",
  "msg_id": "abc12345",
  "role": "Client",
  "timestamp": 1234567890
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `type` | string | ✅ | One of: `heartbeat`, `keepalive`, `elect`, `gateway_announce`, `ack` |
| `node_id` | string | ✅ | Stable for session. Format: `<Platform>_<last4>`, e.g. `Mobile_A914`, `Mac_7F3C`, `iPhone_B21E` |
| `msg_id` | string | ✅ | 8-hex random per message; used for ACK matching |
| `role` | string | ✅ | `Client`, `Gateway`, `Candidate` |
| `timestamp` | number | ✅ | Unix seconds at send time |

## Message types

### `heartbeat`

Broadcast every 2s by every active node. Advertises presence + metrics.

```json
{
  "type": "heartbeat",
  "node_id": "Mobile_A914",
  "msg_id": "abc12345",
  "role": "Client",
  "gateway_status": false,
  "metrics": {
    "rssi": -60,
    "battery": 85,
    "hops_to_gateway": 1
  },
  "neighbor_list": ["Mac_7F3C", "iPhone_B21E"],
  "timestamp": 1234567890
}
```

Additional fields:
- `gateway_status` (bool) — `true` only if this node considers itself the current gateway
- `metrics.rssi` (int, dBm) — WiFi signal strength; -30 excellent, -90 weak
- `metrics.battery` (int, 0–100) — percent; Mac reports `100`
- `metrics.hops_to_gateway` (int) — `0` if self is gateway, else hops via mesh
- `neighbor_list` (string[]) — node_ids heard from in the last 10s

### `keepalive`

Unicast reply to heartbeat from the **current gateway** to known clients. Fires every 5s.
Confirms the gateway is still alive. Used to drive the `gatewayLost` event after 15s silence.

```json
{
  "type": "keepalive",
  "node_id": "Mac_7F3C",
  "msg_id": "def67890",
  "role": "Gateway",
  "relay_port": 10808,
  "gateway_ip": "192.168.12.6",
  "timestamp": 1234567890
}
```

### `elect`

Broadcast when a node observes the current gateway has gone silent (>15s no keepalive).
Triggers re-election round.

```json
{
  "type": "elect",
  "node_id": "Mobile_A914",
  "msg_id": "ghi45678",
  "role": "Candidate",
  "score": 0.62,
  "timestamp": 1234567890
}
```

- `score` (float 0–1) — see [Election scoring](#election-scoring).

### `gateway_announce`

Broadcast by the winner after an election round (or when a new gateway boots and has a better score than whatever it hears).

```json
{
  "type": "gateway_announce",
  "node_id": "Mac_7F3C",
  "msg_id": "jkl23456",
  "role": "Gateway",
  "gateway_status": true,
  "relay_port": 10808,
  "gateway_ip": "192.168.12.6",
  "score": 1.0,
  "timestamp": 1234567890
}
```

- `relay_port` (int) — SOCKS5 port the gateway accepts on the LAN
- `gateway_ip` (string) — IPv4 on LAN for the gateway's SOCKS5

### `ack`

Unicast reply to `gateway_announce` (and optionally `elect`). Confirms delivery.

```json
{
  "type": "ack",
  "node_id": "Mobile_A914",
  "msg_id": "jkl23456",
  "role": "Client",
  "timestamp": 1234567890
}
```

- `msg_id` echoes the message being acked.

## Election scoring

Every node computes its own score on each heartbeat:

```
score = 0.4 * rssi_norm + 0.3 * (1 / (hops + 1)) + 0.3 * battery_norm

where:
  rssi_norm    = clamp((rssi + 100) / 70, 0, 1)   // -100 dBm → 0, -30 dBm → 1
  battery_norm = battery / 100
```

**Mac always scores `1.0`** — plugged in, wired internet, hops=0.

### Tie-breaking

If two nodes compute the same score, the one with the **lexicographically smaller `node_id`** wins.

### Election flow

```
t=0      node X observes no keepalive for 15s
t=0      X broadcasts elect(score=X.score)
t=0–1    other nodes broadcast elect() with their own scores
t=1.5    each node picks max(seen_scores) as winner
t=1.5    winner broadcasts gateway_announce()
t=1.5–2  losers send ack to winner
t=2      steady state: winner sends keepalive every 5s
```

### Known bug: announce/resolve race

`handleGatewayAnnounce()` marks the sender as gateway *immediately*, but `resolveElection()` fires 300ms later and overrides that decision with whatever the local tally decided. Fix (Chat 09):

- On `gateway_announce` with `score > local.bestElectScore`: update `bestElectScore`/`bestElectNodeId` to the remote values AND set `electionInProgress = false` so `resolveElection()` becomes a no-op.

This is implemented in both `DiscoveryModule.swift` and `MeshController.kt` but not yet verified on-device. See [ROADMAP.md](ROADMAP.md) bug #1.

## Rate limits

To prevent ACK storms observed in early testing:

- `heartbeat`: max 1 per 2s per node
- `keepalive`: max 1 per 5s per (gateway, client) pair
- `elect`: max 3 per minute per node
- `gateway_announce`: max 1 per 10s per node
- `ack`: no rate limit (but only in response to announce/elect)

## JS events (React Native)

Surfaced to the RN layer via `DeviceEventEmitter` / `NativeEventEmitter`:

| Event | Payload | When |
|-------|---------|------|
| `gatewayChanged` | `{ nodeId, ip, port }` | A new gateway was elected (or first heard) |
| `becomeGateway` | `{ nodeId }` | This node won an election |
| `gatewayLost` | `{ lastNodeId }` | No keepalive for 15s, election triggered |
| `vpnStateChanged` | `{ state }` | Local VPN state changed: idle/connecting/connected/failed |
| `nodesChanged` | `{ nodes: NodeInfo[] }` | Neighbor list changed (add/remove/metric update) |

Subscribed in `src/services/VpnService.ts`, bridged to Zustand in `src/store/useVPNStore.ts`.

## Versioning

- **v1** (current) — unencrypted JSON. **Do not ship to Iran.** A passive WiFi sniffer sees the whole mesh topology.
- **v2** (planned) — wrap each datagram in ChaCha20-Poly1305 with a pre-shared key derived from daily rotating SSID+password. Adds 28 bytes overhead (12 nonce + 16 tag).
- **v3** (speculative) — full Noise protocol handshake between nodes; per-session keys.

When bumping versions, add a `v` field to the envelope; nodes reject messages with a `v` they don't understand.
