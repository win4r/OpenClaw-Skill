---
name: openclaw
description: Comprehensive guide for installing, configuring, operating, and troubleshooting OpenClaw — a self-hosted, multi-channel AI agent gateway. Use when the user asks about OpenClaw setup, configuration, channel management (WhatsApp/Telegram/Discord/Slack/iMessage/etc.), model provider setup, Gateway operations, multi-agent routing, security hardening, troubleshooting, or any maintenance task related to their local OpenClaw installation. Also use when encountering errors from `openclaw` CLI commands or the Gateway daemon.
---

# OpenClaw Maintenance Skill

OpenClaw is a self-hosted, open-source (MIT) gateway that routes AI agents across WhatsApp, Telegram, Discord, Slack, iMessage, Signal, and 15+ other channels simultaneously. It runs on macOS, Linux, or Windows.

## Quick Reference

### Key Paths

| Path | Purpose |
|---|---|
| `~/.openclaw/openclaw.json` | Main config (JSON5) |
| `~/.openclaw/.env` | Global env fallback |
| `~/.openclaw/workspace` | Default agent workspace |
| `~/.openclaw/agents/<id>/` | Per-agent state + sessions |
| `OPENCLAW_CONFIG_PATH` | Override config location |
| `OPENCLAW_STATE_DIR` | Override state directory |
| `OPENCLAW_HOME` | Override home directory |

### Essential Commands

```
openclaw status                    # Overall status
openclaw gateway status            # Gateway daemon status
openclaw gateway status --deep     # Deep scan including system services
openclaw doctor                    # Diagnose config/service issues
openclaw doctor --fix              # Auto-fix safe issues
openclaw logs --follow             # Tail gateway logs
openclaw channels status --probe   # Channel health check
openclaw security audit            # Security posture check
openclaw security audit --fix      # Auto-fix security issues
```

### Default Gateway

- Bind: `127.0.0.1:18789` (loopback)
- Dashboard: `http://127.0.0.1:18789/`
- Protocol: WebSocket (JSON text frames)

## Core Workflow

### Diagnosing Issues

Always follow this command ladder:

1. `openclaw status` — quick overview
2. `openclaw gateway status` — daemon running? RPC probe ok?
3. `openclaw logs --follow` — watch for errors
4. `openclaw doctor` — config/service diagnostics
5. `openclaw channels status --probe` — per-channel health

### Starting / Restarting Gateway

```bash
# Foreground with verbose logging
openclaw gateway --port 18789 --verbose

# Force-kill existing listener then start
openclaw gateway --force

# Service management (launchd on macOS, systemd on Linux)
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
```

### Configuration

Edit config via any method:

```bash
# Interactive wizard
openclaw onboard                    # Full setup
openclaw configure                  # Config wizard

# CLI one-liners
openclaw config get <path>          # Read value
openclaw config set <path> <value>  # Set value (JSON5 or raw string)
openclaw config unset <path>        # Remove value

# Direct edit
# Edit ~/.openclaw/openclaw.json (JSON5 format)
# Gateway hot-reloads on save (if gateway.reload.mode != "off")
```

Minimal config example:

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

### Channel Setup

For detailed per-channel setup, see [references/channels.md](references/channels.md).
For per-channel troubleshooting (failure signatures, setup walkthroughs), see [references/channel_troubleshooting.md](references/channel_troubleshooting.md).

Quick channel add:

```bash
# Interactive wizard
openclaw channels add

# Non-interactive
openclaw channels add --channel telegram --account default --name "My Bot" --token $BOT_TOKEN
openclaw channels login --channel whatsapp     # QR pairing for WhatsApp
openclaw channels status --probe               # Verify
```

### Model Provider Setup

For detailed provider setup, see [references/providers.md](references/providers.md).

```bash
# Set default model
openclaw models set anthropic/claude-sonnet-4-5

# List available models
openclaw models list --all

# Check auth/token status
openclaw models status --probe

# Add auth interactively
openclaw models auth add
```

Config example:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-5",
        fallbacks: ["openai/gpt-5.2"],
      },
    },
  },
}
```

### Multi-Agent Routing

For detailed multi-agent config, see [references/multi_agent.md](references/multi_agent.md).

```bash
openclaw agents add <id>                # Create agent
openclaw agents list --bindings         # Show agent-channel bindings
openclaw agents delete <id>             # Remove agent
```

### Security

For detailed security hardening, see [references/security.md](references/security.md).
For secrets management (SecretRef, vault integration), see [references/secrets.md](references/secrets.md).
For sandboxing (Docker isolation for tools), see [references/sandboxing.md](references/sandboxing.md).
For full config field reference, see [references/config_reference.md](references/config_reference.md).

```bash
openclaw security audit                 # Check posture
openclaw security audit --deep          # Live gateway probe
openclaw security audit --fix           # Auto-fix safe issues
openclaw secrets reload                 # Re-resolve secret refs
openclaw secrets audit                  # Scan for plaintext leaks
```

### Update / Uninstall

```bash
# Update
npm install -g openclaw@latest
openclaw doctor          # Run after update to apply migrations

# Uninstall
openclaw uninstall
```

## Tools Reference

OpenClaw provides built-in tools to the AI agent. Manage with `tools.allow`, `tools.deny`, and `tools.profile` in config.

**Tool profiles**: `minimal`, `coding`, `messaging`, `full` (default).

**Tool groups** (for allow/deny):
- `group:runtime` — exec, bash, process
- `group:fs` — read, write, edit, apply_patch
- `group:sessions` — sessions_list/history/send/spawn, session_status
- `group:memory` — memory_search, memory_get
- `group:web` — web_search, web_fetch
- `group:ui` — browser, canvas
- `group:automation` — cron, gateway
- `group:messaging` — message
- `group:nodes` — nodes

## Common Failure Signatures

| Error | Cause | Fix |
|---|---|---|
| `refusing to bind gateway ... without auth` | Non-loopback bind without token | Set `gateway.auth.token` or `gateway.auth.password` |
| `another gateway instance is already listening` / `EADDRINUSE` | Port conflict | `openclaw gateway --force` or change port |
| `Gateway start blocked: set gateway.mode=local` | Local mode not enabled | Set `gateway.mode="local"` |
| `unauthorized` / reconnect loop | Token/password mismatch | Check `OPENCLAW_GATEWAY_TOKEN` or config auth |
| `device identity required` | Missing device auth | Ensure client completes connect.challenge flow |
| No replies from bot | Pairing/allowlist/mention gating | Check `openclaw pairing list`, DM policy, mention patterns |

## Environment Variables

| Variable | Purpose |
|---|---|
| `OPENCLAW_GATEWAY_TOKEN` | Gateway auth token |
| `OPENCLAW_GATEWAY_PASSWORD` | Gateway auth password |
| `OPENCLAW_GATEWAY_PORT` | Override gateway port |
| `OPENCLAW_CONFIG_PATH` | Override config file path |
| `OPENCLAW_STATE_DIR` | Override state directory |
| `OPENCLAW_HOME` | Override home directory |
| `OPENCLAW_LOAD_SHELL_ENV` | Import shell env (set to `1`) |
| `BRAVE_API_KEY` | For web_search tool |
