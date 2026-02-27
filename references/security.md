# OpenClaw Security Reference

## Table of Contents

- [Security Model](#security-model)
- [Quick Audit](#quick-audit)
- [Gateway Authentication](#gateway-authentication)
- [DM Access Control](#dm-access-control)
- [Group Chat Security](#group-chat-security)
- [Tool Access Control](#tool-access-control)
- [Sandboxing](#sandboxing)
- [Secrets Management](#secrets-management)
- [Network Hardening](#network-hardening)
- [Hardened Baseline Config](#hardened-baseline-config)
- [Incident Response](#incident-response)

## Security Model

OpenClaw is a **personal assistant** — one user/trust boundary per gateway.

Key principles:
- One trusted operator per gateway instance
- Not designed for multi-tenant isolation between adversarial users
- For multiple untrusted users, use separate gateways (ideally separate hosts)
- Gateway authenticated callers are trusted at gateway scope

## Quick Audit

```bash
openclaw security audit            # Check config + local state
openclaw security audit --deep     # Live Gateway probe
openclaw security audit --fix      # Auto-tighten safe defaults + chmod
openclaw security audit --json     # JSON output for scripting
```

The audit checks:
- Who can talk to your bot (DM policy, allowlists)
- Where the bot can act (bind, network exposure)
- What the bot can touch (tools, filesystem, exec)

## Gateway Authentication

Auth is **required by default** when binding to non-loopback.

### Config

```json5
{
  gateway: {
    auth: {
      mode: "token",                        // "token" or "password"
      token: "replace-with-long-random-token",
      // OR
      password: "your-password",
    },
  },
}
```

### Environment Variables

```bash
OPENCLAW_GATEWAY_TOKEN=your-token
OPENCLAW_GATEWAY_PASSWORD=your-password
```

### Error: `refusing to bind gateway ... without auth`

Fix: Set `gateway.auth.token` or `gateway.auth.password` in config/env.

## DM Access Control

Per-channel `dmPolicy` setting:

| Policy | Behavior | Use Case |
|---|---|---|
| `"pairing"` | Unknown senders get pairing code to approve | Default, recommended |
| `"allowlist"` | Only senders in `allowFrom` list | Strict, known users only |
| `"open"` | Allow all DMs (needs `allowFrom: ["*"]`) | Public bots (risky) |
| `"disabled"` | Ignore all DMs | Group-only channels |

Session isolation for multi-user DMs:

```json5
{
  session: {
    dmScope: "per-channel-peer",    // Isolate sessions per sender
    // OR "per-account-channel-peer" for multi-account channels
  },
}
```

## Group Chat Security

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },    // All groups need @mention
      },
    },
  },
  agents: {
    list: [{
      id: "main",
      groupChat: {
        mentionPatterns: ["@openclaw", "openclaw"],
      },
    }],
  },
}
```

## Tool Access Control

### Tool Profiles

| Profile | Allowed |
|---|---|
| `minimal` | session_status only |
| `coding` | group:fs, group:runtime, group:sessions, group:memory, image |
| `messaging` | group:messaging, sessions tools |
| `full` | No restrictions (default) |

### Deny Dangerous Tools

```json5
{
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs",
           "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

### Per-Agent Tool Override

```json5
{
  agents: {
    list: [{
      id: "support",
      tools: {
        profile: "messaging",
        allow: ["slack"],
      },
    }],
  },
}
```

### Provider-Specific Tool Policy

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
    },
  },
}
```

## Sandboxing

Recommended for tool-enabled agents. See docs: https://docs.openclaw.ai/gateway/sandboxing

## Secrets Management

### SecretRef Pattern

Avoid plaintext secrets in config — use refs:

```json5
{
  models: {
    providers: {
      openai: {
        apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
      },
    },
  },
}
```

Sources: `env`, `file`, `exec`.

### CLI

```bash
openclaw secrets reload             # Re-resolve refs, swap runtime snapshot
openclaw secrets audit              # Scan for plaintext, unresolved refs
openclaw secrets configure          # Interactive setup + SecretRef mapping
openclaw secrets apply --from <f>   # Apply plan (--dry-run supported)
```

## Network Hardening

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",               // Only localhost
    // bind options: loopback | tailnet | lan | auto | custom
  },
}
```

Remote access options:
- **Preferred**: Tailscale or VPN
- **Alternative**: SSH tunnel — `ssh -N -L 18789:127.0.0.1:18789 user@host`

## Hardened Baseline Config

Copy/paste for maximum lockdown:

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: {
      mode: "token",
      token: "replace-with-long-random-token",
    },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs",
           "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

## Incident Response

### 1. Contain

```bash
openclaw gateway stop                   # Stop gateway
openclaw channels logout --channel <c>  # Logout channels
```

### 2. Rotate Secrets

Assume compromise if secrets leaked. Rotate all API keys, tokens, passwords.

### 3. Audit

```bash
openclaw security audit --deep
openclaw secrets audit
openclaw logs --limit 1000              # Review recent logs
```

### 4. Collect for Report

```bash
openclaw status --all --json > status.json
openclaw logs --json > logs.json
```
