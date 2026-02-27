# OpenClaw Configuration Reference

Complete field-by-field reference for `~/.openclaw/openclaw.json`. This covers the most important configuration sections.

## Table of Contents

- [Config File](#config-file)
- [Gateway Section](#gateway-section)
- [Channels Section](#channels-section)
- [Agent Defaults](#agent-defaults)
- [Tools Section](#tools-section)
- [Session Section](#session-section)
- [Multi-Agent Routing](#multi-agent-routing)
- [Secrets Section](#secrets-section)
- [Environment Section](#environment-section)
- [Browser Section](#browser-section)
- [Plugins Section](#plugins-section)
- [Hooks Section](#hooks-section)
- [Cron Section](#cron-section)
- [Config Includes](#config-includes)

## Config File

Path: `~/.openclaw/openclaw.json` (JSON5 format)

Override via `OPENCLAW_CONFIG_PATH` env var.

## Gateway Section

```json5
{
  gateway: {
    mode: "local",               // "local" (run gateway) | "remote" (connect to remote)
    port: 18789,                 // Single multiplexed port for WS + HTTP
    bind: "loopback",           // "auto" | "loopback" (default) | "lan" | "tailnet" | "custom"
    auth: {
      mode: "token",            // "none" | "token" | "password" | "trusted-proxy"
      token: "your-token",
      // password: "your-password",
      allowTailscale: true,     // Allow Tailscale Serve identity headers
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,   // Default true
      },
    },
    tailscale: {
      mode: "off",              // "off" | "serve" | "funnel"
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
    },
    remote: {                   // For mode: "remote"
      url: "ws://gateway.tailnet:18789",
      transport: "ssh",         // "ssh" | "direct"
      token: "your-token",
    },
    trustedProxies: [],         // IP addresses of trusted reverse proxies
    reload: {
      mode: "hybrid",          // "off" | "hot" | "restart" | "hybrid"
    },
  },
}
```

**Port precedence** (highest to lowest):
1. `--port` CLI flag
2. `OPENCLAW_GATEWAY_PORT` env var
3. `gateway.port` in config
4. Default: `18789`

**Auth notes:**
- Non-loopback binds REQUIRE auth
- `"trusted-proxy"`: delegate to identity-aware reverse proxy
- Rate limiter blocks per client IP, returns `429 + Retry-After`

## Channels Section

### Per-Channel Config

```json5
{
  channels: {
    whatsapp: {
      enabled: true,
      dmPolicy: "pairing",      // "pairing" | "allowlist" | "open" | "disabled"
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",  // "open" | "allowlist" | "disabled"
      groupAllowFrom: ["+15551234567"],
      groups: {
        "*": { requireMention: true },
        "123@g.us": { requireMention: false },
      },
      selfChatMode: true,       // For personal number use
      textChunkLimit: 4096,
      sendReadReceipts: true,
    },
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      allowFrom: ["tg:123"],
      streaming: "off",         // "off" | "partial" | "block" | "progress"
      proxy: "socks5://user:pass@host:1080",
    },
    discord: {
      enabled: true,
      token: "bot-token",
      dmPolicy: "pairing",
      guilds: {
        SERVER_ID: {
          requireMention: true,
          users: ["USER_ID"],
        },
      },
    },
    slack: {
      enabled: true,
      mode: "socket",           // "socket" | "http"
      appToken: "xapp-...",
      botToken: "xoxb-...",
      signingSecret: "...",     // For HTTP mode
    },
  },
}
```

### Multi-Account (All Channels)

```json5
{
  channels: {
    telegram: {
      accounts: {
        personal: { botToken: "...", dmPolicy: "pairing" },
        work: { botToken: "...", dmPolicy: "allowlist", allowFrom: ["tg:456"] },
      },
    },
  },
}
```

## Agent Defaults

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: {
        primary: "anthropic/claude-sonnet-4-5",
        fallbacks: ["openai/gpt-5.2"],
      },
      imageModel: {
        primary: "openai/dall-e-3",
      },
      models: {                 // Model catalog + allowlist for /model
        "anthropic/claude-sonnet-4-5": { alias: "Sonnet" },
        "openai/gpt-5.2": { alias: "GPT" },
      },
      imageMaxDimensionPx: 1200,  // Image downscaling (reduces vision-token usage)
      heartbeat: {
        every: "30m",
      },
      compaction: { ... },
      contextPruning: { ... },
      sandbox: {
        mode: "off",             // "off" | "non-main" | "all"
        scope: "session",        // "session" | "agent" | "shared"
        workspaceAccess: "none", // "none" | "ro" | "rw"
      },
    },
  },
}
```

## Tools Section

```json5
{
  tools: {
    profile: "full",            // "minimal" | "coding" | "messaging" | "full"
    allow: [],                  // Explicit tool allow (if non-empty, everything else blocked)
    deny: ["browser", "canvas"], // Block specific tools
    byProvider: {               // Per-model tool restrictions
      "google-antigravity": { profile: "minimal" },
    },
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
      },
    },
    exec: {
      security: "allow",       // "allow" | "ask" | "deny"
      ask: "risky",            // "always" | "risky" | "never"
      backgroundMs: 10000,
      timeoutSec: 1800,
    },
    fs: {
      workspaceOnly: false,    // Restrict to workspace directory
    },
    web: {
      search: {
        apiKey: "${BRAVE_API_KEY}",
      },
    },
  },
}
```

### Tool Groups

| Group | Tools Included |
|---|---|
| `group:runtime` | exec, process, bash |
| `group:fs` | read, write, edit, apply_patch |
| `group:sessions` | sessions_list, sessions_history, sessions_send, sessions_spawn, session_status |
| `group:memory` | memory_search, memory_get |
| `group:web` | web_search, web_fetch |
| `group:ui` | browser, canvas |
| `group:automation` | cron, gateway |
| `group:messaging` | message |
| `group:nodes` | nodes |

## Session Section

```json5
{
  session: {
    dmScope: "main",           // "main" | "per-channel-peer" | "per-account-channel-peer"
    mainKey: "main",           // Key for the main session
    historyLimit: 100,
  },
}
```

## Multi-Agent Routing

```json5
{
  agents: {
    list: [
      { id: "main", workspace: "~/.openclaw/workspace" },
      { id: "work", workspace: "~/.openclaw/workspace-work",
        model: { primary: "anthropic/claude-opus-4-6" },
        tools: { profile: "coding" },
      },
    ],
  },
  bindings: [
    { agentId: "work", match: { channel: "telegram", accountId: "work" } },
    { agentId: "main", match: { channel: "whatsapp" } },
  ],
}
```

### Binding Match Fields

- `channel` — channel name
- `accountId` — channel account ID
- `peer.kind` — `"direct"` or `"group"`
- `peer.id` — specific sender/group ID

## Secrets Section

See [references/secrets.md](secrets.md) for full details.

## Environment Section

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
    shellEnv: {
      enabled: true,           // Import shell env
      timeoutMs: 15000,
    },
  },
}
```

## Browser Section

```json5
{
  browser: {
    executablePath: "/usr/bin/google-chrome",
    // ... CDP config
  },
}
```

## Plugins Section

```json5
{
  plugins: {
    allow: ["backups", "memory-lancedb-pro"],  // Explicit allowlist
  },
}
```

## Hooks Section

```json5
{
  hooks: {
    gmail: { ... },
    // Webhook integrations
  },
}
```

## Cron Section

```json5
{
  cron: {
    jobs: [
      { schedule: "0 9 * * *", message: "Good morning!", channel: "whatsapp" },
    ],
  },
}
```

## Config Includes

Split config across files using `$include`:

```json5
{
  $include: ["./channels.json", "./agents.json"],
  gateway: { ... },
}
```

Env var substitution works inside `$include` files.
