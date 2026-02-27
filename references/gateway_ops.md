# OpenClaw Gateway Operations Reference

## Table of Contents

- [Architecture](#architecture)
- [Starting the Gateway](#starting-the-gateway)
- [Service Management](#service-management)
- [Port and Bind Configuration](#port-and-bind-configuration)
- [Hot Reload](#hot-reload)
- [Remote Access](#remote-access)
- [Multiple Gateways](#multiple-gateways)
- [Health Checks](#health-checks)
- [Operator Commands](#operator-commands)
- [Gateway Protocol](#gateway-protocol)
- [Install / Update / Uninstall](#install--update--uninstall)
- [Troubleshooting](#troubleshooting)

## Architecture

- Single long-lived daemon owns all messaging surfaces
- One multiplexed port for WebSocket RPC, HTTP APIs, Control UI, and hooks
- One Gateway per host controls a single WhatsApp/Baileys session
- Canvas served at `/__openclaw__/canvas/` and `/__openclaw__/a2ui/` on same port
- Default bind: `127.0.0.1:18789` (loopback)

Components:
- **Gateway daemon**: provider connections, typed WS API, JSON Schema validation
- **Clients** (macOS app / CLI / web admin): one WS connection each
- **Nodes** (macOS / iOS / Android / headless): WS with `role: node`, expose device commands
- **WebChat**: static UI using Gateway WS API

## Starting the Gateway

```bash
# Foreground (dev/debug)
openclaw gateway --port 18789
openclaw gateway --port 18789 --verbose     # Debug/trace to stdout
openclaw gateway --force                    # Kill existing listener first

# With auth
openclaw gateway --token <token>
openclaw gateway --password <password>

# With Tailscale
openclaw gateway --tailscale serve          # Tailscale Serve
openclaw gateway --tailscale funnel         # Tailscale Funnel (public)

# Dev profile (separate port 19001)
openclaw --dev gateway --allow-unconfigured
```

## Service Management

### macOS (launchd)

```bash
openclaw gateway install        # Install launch agent (ai.openclaw.gateway)
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway status         # Probe RPC
openclaw gateway uninstall
```

### Linux (systemd user)

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway.service
openclaw gateway status
```

### Linux (system service)

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway.service
sudo loginctl enable-linger <user>      # Keep user service running
```

### Service Install Options

```bash
openclaw gateway install --port <port> --runtime <node|bun> --token <token> --force
```

Note: Node runtime recommended; Bun has WhatsApp/Telegram bugs.

## Port and Bind Configuration

Precedence (highest to lowest):
1. `--port` CLI flag
2. `OPENCLAW_GATEWAY_PORT` env var
3. `gateway.port` in config
4. Default: `18789`

Bind modes:
- `loopback` (default) — localhost only
- `tailnet` — Tailscale network only
- `lan` — local network
- `auto` — auto-detect
- `custom` — manual bind address

## Hot Reload

Config setting: `gateway.reload.mode`

| Mode | Behavior |
|---|---|
| `off` | No automatic reload |
| `hot` | Hot-apply supported changes without restart |
| `restart` | Full restart on config change |
| `hybrid` | Hot-apply what it can, restart for the rest |

What hot-applies: system prompt changes, model selection, tool config.
What needs restart: port/bind changes, channel add/remove, auth changes.

## Remote Access

### SSH Tunnel (simple)

```bash
ssh -N -L 18789:127.0.0.1:18789 user@host
```

Then connect clients to `ws://127.0.0.1:18789` with same token/password.

### Tailscale

```bash
openclaw gateway --tailscale serve      # Private (tailnet only)
openclaw gateway --tailscale funnel     # Public (internet-accessible)
```

### VPN

Any VPN that provides network-level access. Gateway auth still required.

## Multiple Gateways

Requirements per instance:
- Unique `gateway.port`
- Unique `OPENCLAW_CONFIG_PATH`
- Unique `OPENCLAW_STATE_DIR`
- Unique `agents.defaults.workspace`

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

## Health Checks

### Liveness

Open WS → send `connect` → expect `hello-ok` response.

### Readiness

```bash
openclaw gateway status             # Runtime: running, RPC probe: ok
openclaw channels status --probe    # Connected/ready channels
openclaw health                     # Overall health check
```

### Operational Checks

```bash
openclaw gateway status --deep      # System-level scan
openclaw gateway status --json      # JSON output for scripting
openclaw doctor                     # Full diagnostics
openclaw doctor --fix               # Auto-repair
```

## Operator Commands

```bash
openclaw gateway status [--deep] [--json]
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw logs --limit 200
openclaw logs --json
openclaw doctor
```

## Gateway Protocol

- Transport: WebSocket, text frames, JSON payloads
- First frame must be `connect`
- After handshake: `hello-ok` snapshot (presence, health, stateVersion, uptimeMs)
- Requests: `{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
- Events: `{type:"event", event, payload, seq?, stateVersion?}`
- Auth token via `connect.params.auth.token` or `OPENCLAW_GATEWAY_TOKEN`
- Idempotency keys required for `send`, `agent` methods
- Nodes declare `role: "node"` with capabilities in `connect`

Common events: `connect.challenge`, `agent`, `chat`, `presence`, `tick`, `health`, `heartbeat`, `shutdown`.

## Install / Update / Uninstall

### Install Methods

```bash
# Installer script (macOS / Linux)
curl -fsSL https://openclaw.ai/install.sh | bash

# npm
npm install -g openclaw@latest
openclaw onboard --install-daemon

# From source
git clone https://github.com/openclaw/openclaw.git
cd openclaw && pnpm install && pnpm ui:build && pnpm build
pnpm link --global
openclaw onboard --install-daemon
```

System requirements: Node 22+, macOS/Linux/Windows.

### Update

```bash
npm install -g openclaw@latest
openclaw doctor              # Apply any migrations
```

### Uninstall

```bash
openclaw uninstall
```

## Troubleshooting

### Gateway Service Not Running

```bash
openclaw gateway status --deep
openclaw doctor
openclaw logs --follow
```

Common errors:
- `Runtime: stopped` — check exit hints
- `Config (cli) vs Config (service)` mismatch — re-install service
- Port conflict — `openclaw gateway --force`

### Dashboard/UI Not Loading

```bash
openclaw gateway status --json       # Check probe URL
openclaw doctor
```

Check: correct URL, auth mode/token match, device identity flow.

### After Upgrade Issues

Run `openclaw doctor` immediately after updating. Common breaking changes:
1. Auth/URL override behavior changes
2. Bind/auth guardrails stricter
3. Pairing/device identity state changes

### Browser Tool Fails

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw doctor
```

Check: valid browser path, CDP profile reachability, Chrome extension relay.

### Cron/Heartbeat Not Firing

```bash
openclaw cron status
openclaw system heartbeat last
```

Verify Gateway is running and cron jobs are enabled.
