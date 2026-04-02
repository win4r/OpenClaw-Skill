# Plugins (Extensions) Reference

Plugins extend OpenClaw with new channels, tools, CLI commands, RPC methods, background services, and skills.

## Quick Start

```bash
openclaw plugins list                          # List loaded plugins
openclaw plugins install @openclaw/voice-call  # Install from npm
# Restart the Gateway, then configure under plugins.entries.<id>.config.
```

## Available Plugins (Official)

| Plugin | Package | Description |
|---|---|---|
| Memory (Core) | bundled | Memory search plugin (enabled by default via `plugins.slots.memory`) |
| Memory (LanceDB) | bundled | Long-term memory with auto-recall/capture (`plugins.slots.memory = "memory-lancedb"`) |
| [Voice Call](https://docs.openclaw.ai/plugins/voice-call) | `@openclaw/voice-call` | Voice calls via Twilio |
| [Zalo Personal](https://docs.openclaw.ai/plugins/zalouser) | `@openclaw/zalouser` | Zalo personal messaging |
| [Matrix](https://docs.openclaw.ai/channels/matrix) | `@openclaw/matrix` | Matrix channel |
| [Nostr](https://docs.openclaw.ai/channels/nostr) | `@openclaw/nostr` | Nostr channel |
| [Zalo](https://docs.openclaw.ai/channels/zalo) | `@openclaw/zalo` | Zalo channel |
| [Microsoft Teams](https://docs.openclaw.ai/channels/msteams) | `@openclaw/msteams` | MS Teams (plugin-only since 2026.1.15) |
| Google Antigravity OAuth | bundled (`google-antigravity-auth`) | Provider auth (disabled by default) |
| Gemini CLI OAuth | bundled (`google-gemini-cli-auth`) | Provider auth (disabled by default) |
| Qwen OAuth | bundled (`qwen-portal-auth`) | Provider auth (disabled by default) |
| Copilot Proxy | bundled | Local VS Code Copilot Proxy bridge (disabled by default) |

## Plugin Capabilities

Plugins can register:
- Gateway RPC methods
- Gateway HTTP handlers / routes
- Agent tools
- CLI commands
- Background services
- Optional config validation
- Skills (via plugin manifest `skills` directories)
- Auto-reply commands (execute without invoking AI agent)
- Capabilities (see below)

### Capability Registration Model

Plugins register against specific capability types:

| Capability | Registration Method | Examples |
|---|---|---|
| Text inference | `api.registerProvider(...)` | openai, anthropic |
| CLI backend | `api.registerCliBackend(...)` | openai, anthropic |
| Speech (TTS/STT) | `api.registerSpeechProvider(...)` | elevenlabs, microsoft |
| Media understanding | `api.registerMediaUnderstandingProvider(...)` | openai, google |
| Image generation | `api.registerImageGenerationProvider(...)` | openai, google |
| Web search | `api.registerWebSearchProvider(...)` | google |
| Messaging/Channel | `api.registerChannel(...)` | msteams, matrix |

### Plugin Shapes

| Shape | Description |
|---|---|
| `plain-capability` | Registers exactly one capability type |
| `hybrid-capability` | Registers multiple capability types |
| `hook-only` | Registers only hooks, no capabilities |
| `non-capability` | Registers tools, commands, or services but no capabilities |

Use `openclaw plugins inspect <id>` to view a plugin's shape.

### Context Engine Plugins

Plugins can register custom context engines for controlling how OpenClaw builds model context:

```ts
api.registerContextEngine("my-engine", () => ({
  info: { id: "my-engine", name: "My Context Engine", ownsCompaction: true },
  async ingest() { return { ingested: true }; },
  async assemble({ messages }) { return { messages, estimatedTokens: 0 }; },
  async compact() { return { ok: true, compacted: false }; },
}));
```

Select via: `plugins.slots.contextEngine: "my-engine"`

## Plugin IDs

- **Package packs**: `package.json` `name` field
- **Standalone file**: file base name (`~/.../voice-call.ts` → `voice-call`)

## Config

```json5
{
  plugins: {
    enabled: true,                        // master toggle (default: true)
    allow: ["voice-call"],               // allowlist (optional)
    deny: ["untrusted-plugin"],          // denylist (deny wins)
    load: {
      paths: ["~/Projects/oss/voice-call-extension"],  // extra plugin paths
    },
    entries: {
      "voice-call": {
        enabled: true,
        config: { provider: "twilio" },
      },
    },
  },
}
```

Notes:
- Unknown plugin ids in `entries`, `allow`, `deny`, or `slots` are errors.
- Unknown `channels.<id>` keys are errors unless a plugin manifest declares the channel id.
- Plugin config is validated using the JSON Schema in `openclaw.plugin.json` (`configSchema`).
- If a plugin is disabled, its config is preserved and a warning is emitted.

## Plugin Slots (Exclusive Categories)

```json5
{
  plugins: {
    slots: {
      memory: "memory-core",    // or "none" to disable memory plugins
    },
  },
}
```

Only one plugin per slot. Plugins declare their slot via `kind: "memory"` in the manifest.

## Control UI (Schema + Labels)

Plugins can provide `uiHints` for the Control UI:

```json5
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": { "type": "string" },
      "region": { "type": "string" }
    }
  },
  "uiHints": {
    "apiKey": { "label": "API Key", "sensitive": true },
    "region": { "label": "Region", "placeholder": "us-east-1" }
  }
}
```

## CLI

```bash
openclaw plugins list                              # List loaded plugins
openclaw plugins info <id>                         # Plugin details
openclaw plugins install <path>                    # Install from local path
openclaw plugins install ./extensions/voice-call   # Relative path
openclaw plugins install ./plugin.tgz              # From tarball
openclaw plugins install ./plugin.zip              # From zip
openclaw plugins install -l ./extensions/voice-call # Link (no copy) for dev
openclaw plugins install @openclaw/voice-call      # Install from npm
openclaw plugins install @openclaw/voice-call --pin # Pin exact version
openclaw plugins update <id>                       # Update single plugin
openclaw plugins update --all                      # Update all
openclaw plugins enable <id>                       # Enable plugin
openclaw plugins disable <id>                      # Disable plugin
openclaw plugins doctor                            # Check plugin health
```

`plugins update` writes to `plugins.installs`. Use `--yes` for non-interactive.

## Plugin API (Overview)

A plugin can be:
- A function: `(api) => { ... }`
- An object: `{ id, name, configSchema, register(api) { ... } }`

## Runtime Helpers

Plugins access core helpers through `api.runtime`:

```javascript
// TTS
const result = await api.runtime.tts.textToSpeech({ text: "Hello", cfg: api.config });
const telephony = await api.runtime.tts.textToSpeechTelephony({ text: "Hello", cfg: api.config });
const voices = await api.runtime.tts.listVoices(cfg);

// Media Understanding
await api.runtime.mediaUnderstanding.describeImageFile({ path, model });
await api.runtime.mediaUnderstanding.describeVideoFile({ path, model });
await api.runtime.mediaUnderstanding.transcribeAudioFile({ path, model });

// Subagent (requires plugins.entries.<id>.subagent.allowModelOverride: true)
await api.runtime.subagent.run({ task, model });

// Web Search
const providers = await api.runtime.webSearch.listProviders();
const results = await api.runtime.webSearch.search({ query, provider });

// Image Generation
const image = await api.runtime.imageGeneration.generate({ prompt, model });
const imgProviders = await api.runtime.imageGeneration.listProviders();
```

## Naming Conventions

- Gateway methods: `pluginId.action` (e.g., `voicecall.status`)
- Tools: `snake_case` (e.g., `voice_call`)
- CLI commands: kebab or camel, avoid clashing with core commands

## Skills in Plugins

Plugins can declare skills via `skills/<name>/SKILL.md` in the plugin directory. Skills are loaded/unloaded with `plugins.entries.<id>.enabled`.

## Provider Runtime Hooks

Plugins get access to 24+ provider hooks covering the full model lifecycle. Key hooks:

| Hook | Purpose |
|---|---|
| `catalog` | Provide model catalog |
| `before_model_resolve` | Override provider/model before lookup |
| `before_prompt_build` | Modify system prompt |
| `before_tool_call` | Intercept tool calls (block/approve) |
| `after_tool_call` | Process tool results |
| `tool_result_persist` | Transform tool results before transcript |
| `message_received` / `message_sent` | Message flow hooks |
| `session_start` / `session_end` | Session lifecycle |
| `gateway_start` / `gateway_stop` | Gateway lifecycle |

### Gateway HTTP Routes

```typescript
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

## Channel Catalog Metadata

Channel plugins can declare catalog metadata for discovery:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "docsPath": "/channels/nextcloud-talk",
      "blurb": "Self-hosted chat via Nextcloud Talk webhook bots.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    }
  }
}
```

External catalogs merge from `~/.openclaw/mpm/catalog.json` or via `OPENCLAW_PLUGIN_CATALOG_PATHS`.

## Distribution (npm)

- Main package: `openclaw` (this repo)
- Plugins: separate npm packages under `@openclaw/*` (e.g., `@openclaw/voice-call`)

Requirements:
- `package.json` must include `openclaw.extensions` with entry files.
- Entry files can be `.js` or `.ts` (jiti loads at runtime).
- `openclaw plugins install <npm-spec>` uses `npm pack`, extracts to `~/.openclaw/extensions/<id>/`.
- Scoped packages are normalized to the unscoped id for `plugins.entries.*`.
- Security: `openclaw plugins install` uses `npm install --ignore-scripts` to prevent lifecycle script execution.

### Package Packs

A single npm package can export multiple plugins:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Every `openclaw.extensions` entry must stay inside the plugin directory after symlink resolution.

## Plugin Manifest (`openclaw.plugin.json`)

A required manifest file for directory plugins:

### Required Fields

- `id` — unique plugin identifier
- `name` — human-readable name
- `version` — semver version

### Optional

- `configSchema` — JSON Schema for plugin config
- `uiHints` — UI labels and hints for config fields
- Plugin slot declaration (`kind`)

## Example Plugin: Voice Call

- Source: `extensions/voice-call`
- Skill: `skills/voice-call`
- CLI: `openclaw voicecall start|status`
- Tool: `voice_call`
- RPC: `voicecall.start`, `voicecall.status`
- Config (twilio): `provider: "twilio"` + `twilio.accountSid/authToken/from`
- Config (dev): `provider: "log"` (no network)

## Safety Notes

- Only install plugins you trust.
- Prefer `plugins.allow` allowlists.
- Restart the Gateway after changes.

## Testing Plugins

- In-repo plugins: Vitest tests under `src/**` (e.g., `src/plugins/voice-call.plugin.test.ts`).
- Published plugins: Run their own CI (lint/build/test) and validate `openclaw.extensions` points at the built entrypoint.
