# Presence & Discovery

> Sources:
> - https://docs.openclaw.ai/concepts/presence
> - https://docs.openclaw.ai/gateway/discovery
> - https://docs.openclaw.ai/gateway/bonjour

## Presence

Lightweight, best-effort view of the Gateway and connected clients.

### Presence Fields

| Field | Description |
|---|---|
| `instanceId` | Stable client identifier (recommended for dedup) |
| `host` | Human-friendly hostname |
| `ip` | Best-effort IP address |
| `version` | Client version string |
| `deviceFamily` / `modelIdentifier` | Hardware info |
| `mode` | `ui`, `webchat`, `cli`, `backend`, `probe`, `test`, `node` |
| `lastInputSeconds` | Elapsed time since user activity |
| `reason` | Source: `self`, `connect`, `node-connected`, `periodic` |
| `ts` | Last update timestamp (ms) |

### Data Sources

1. **Gateway self entry**: Always seeded at startup so UIs show the gateway host
2. **WebSocket connections**: Clients register presence on successful handshake
3. **CLI**: Short one-off commands — `cli` mode is not turned into a presence entry
4. **Node connections**: Nodes with `role: node` create presence entries
5. **System events**: Clients send richer periodic beacons via `system-event` method

### Deduplication

- Entries keyed by `instanceId` (case-insensitive)
- Multiple connections from the same instance are merged
- Missing stable identifiers cause duplicates

### Lifecycle

- **TTL**: entries exceeding 5 minutes are removed
- **Size limit**: max 200 entries (oldest purged first)
- Loopback remote addresses are ignored to prevent overwriting client-reported IPs

### Access

```bash
openclaw system presence              # View connected clients/nodes
```

---

## Discovery & Transports

How clients learn where the Gateway is.

### Discovery Methods

#### 1. Bonjour / mDNS (LAN Only)

Gateway advertises itself on the local network via Bonjour/mDNS. macOS and iOS clients auto-discover Gateway on the same LAN.

Broadcast modes:

| Mode | Info Disclosed | Recommendation |
|---|---|---|
| `minimal` (default) | Basic discovery only | Recommended |
| `full` (opt-in) | Includes `cliPath` and `sshPort` | Trusted networks only |
| `off` | Disables local discovery | Maximum privacy |

See [bonjour.md](bonjour.md) for TXT keys, wide-area DNS-SD, and debugging.

#### 2. Tailnet (Cross-Network)

Gateway registers with Tailscale for cross-network access. Clients discover Gateway via Tailscale MagicDNS.

```json5
{
  gateway: {
    tailscale: {
      mode: "serve",        // "off" | "serve" | "funnel"
      resetOnExit: false,
    },
  },
}
```

#### 3. Manual / SSH Target

Users manually configure the Gateway address or use SSH tunneling:

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

### Transport Selection (Client Policy)

- Direct WS preferred when available (LAN/Tailnet)
- SSH fallback for remote access without Tailscale
- All transports require standard WS handshake + auth tokens

### Responsibilities

| Component | Responsibility |
|---|---|
| Gateway | Advertise via Bonjour, accept WS connections |
| macOS App | Discover via Bonjour/Tailnet, connect via WS |
| iOS App | Discover via Bonjour, connect via WS |
| CLI | Manual config, connect via WS |
