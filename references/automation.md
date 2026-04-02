# OpenClaw Automation Reference (Cron Jobs, Webhooks, Gmail Pub/Sub)

## Table of Contents

- [Cron Jobs](#cron-jobs)
- [Webhooks](#webhooks)
- [Gmail Pub/Sub](#gmail-pubsub)
- [Standing Orders](#standing-orders)
- [Hooks](#hooks)

---

## Cron Jobs

Cron runs inside the **Gateway** (not inside the model). Jobs persist under `~/.openclaw/cron/jobs.json`.

### Quick Start

```bash
# One-shot reminder
openclaw cron add \
  --name "Reminder" \
  --at "2026-02-01T16:00:00Z" \
  --session main \
  --system-event "Reminder: check the docs" \
  --wake now \
  --delete-after-run

# Recurring isolated job with announce
openclaw cron add \
  --name "Morning brief" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize overnight updates." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"

# Management
openclaw cron list
openclaw cron run <job-id>
openclaw cron runs --id <job-id>
```

### Concepts

#### Jobs

Each job has:
- **Schedule** (when it runs)
- **Payload** (what it does)
- **Delivery mode** (optional): `announce`, `webhook`, or `none`
- **Agent binding** (optional): `agentId` — run under a specific agent

#### Schedules

| Type | Description | Example |
|---|---|---|
| `at` | One-shot timestamp (ISO 8601) | `schedule.at: "2026-02-01T16:00:00Z"` |
| `every` | Fixed interval (ms) | `schedule.every: 3600000` |
| `cron` | 5-field cron expression + optional timezone | `schedule.cron: "0 7 * * *"` |

Options: `--stagger 30s` to set stagger window, `--exact` to force `staggerMs = 0`.

#### Execution Modes

| Mode | Session Key | Description |
|---|---|---|
| **Main session** | `main` | System event → heartbeat prompt |
| **Isolated** | `cron:<jobId>` | Dedicated agent turn, fresh session per run |

**Main session jobs**: `payload.kind = "systemEvent"`. Wake options: `"now"` (immediate) or `"next-heartbeat"`.

**Isolated jobs**: Each run starts a fresh sessionId. Prompt prefixed with `[cron:<jobId> <job name>]`.

#### Payload Shapes

- `systemEvent`: main-session only, routed through heartbeat
- `agentTurn`: isolated only, runs a dedicated agent turn with `message`, optional `model`/`thinking`/`timeoutSeconds`

#### Delivery

| Mode | Behavior |
|---|---|
| `announce` (default for isolated) | Deliver summary to target channel + main session summary |
| `webhook` | POST to `delivery.to` URL |
| `none` | Internal only, no delivery |

Delivery fields:
- `delivery.channel`: `whatsapp` / `telegram` / `discord` / `slack` / `mattermost` / `signal` / `imessage` / `last`
- `delivery.to`: channel-specific target or webhook URL
- `delivery.bestEffort`: avoid failing if announce delivery fails

**Telegram topics**: Use `delivery.to: "-1001234567890:topic:123"` format.

#### Model & Thinking Overrides

Override priority: Job payload → Hook-specific defaults → Agent config default.

- `model`: `provider/model` string or alias
- `thinking`: `off`, `minimal`, `low`, `medium`, `high`, `xhigh`

### Advanced Features

#### Lightweight Bootstrap

Use `lightContext: true` in the payload to skip workspace bootstrap file injection — useful for simple chores that don't need full agent context.

#### Tool Allowlists

```bash
openclaw cron add --name "Job" --tools exec,read     # Only allow exec and read
openclaw cron edit <jobId> --clear-tools              # Remove tool restriction
```

#### Agent Pinning

```bash
openclaw cron add --name "Job" --agent ops            # Run under specific agent
openclaw cron edit <jobId> --agent main                # Change agent binding
```

#### Stagger Control

Top-of-hour cron expressions get a deterministic stagger window (up to 5 minutes) to reduce load spikes. Fixed-hour expressions like `0 7 * * *` remain exact.

```bash
openclaw cron add --name "Job" --cron "0 * * * *" --stagger 30s
openclaw cron edit <jobId> --exact                     # Force staggerMs = 0
```

#### Custom Persistent Sessions

Use a named session to maintain context across runs:

```json
{
  "name": "Project monitor",
  "schedule": { "kind": "every", "everyMs": 300000 },
  "sessionTarget": "session:project-alpha-monitor",
  "payload": {
    "kind": "agentTurn",
    "message": "Check project status and update the running log."
  }
}
```

### Retry Policy

**One-shot jobs:**
- Transient errors (rate limit, overloaded, network, 5xx): retry up to 3 times with exponential backoff (30s, 1m, 5m)
- Permanent errors (auth, validation): disable immediately

**Recurring jobs:**
- Any error: exponential backoff (30s, 1m, 5m, 15m, 60m) before next run
- Job stays enabled; backoff resets after successful execution

### Configuration

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    maxConcurrentRuns: 1,
    retry: {
      maxAttempts: 3,
      backoffMs: [60000, 120000, 300000],
      retryOn: ["rate_limit", "overloaded", "network", "server_error"],
    },
    webhookToken: "your-token",
    sessionRetention: "24h",          // Isolation run-session pruning
    runLog: {
      maxBytes: "2mb",
      keepLines: 2000,
    },
  },
}
```

Disable cron: `cron.enabled: false` or env `OPENCLAW_SKIP_CRON=1`.

### Cron Storage & Pruning

- **Job store**: `~/.openclaw/cron/jobs.json` (Gateway-managed, manual edits only safe when Gateway stopped)
- **Run history**: `~/.openclaw/cron/runs/<jobId>.jsonl` (auto-pruned by `runLog.maxBytes` / `runLog.keepLines`)
- **Isolation session pruning**: default `24h` retention for `cron:<jobId>:run:<uuid>` sessions. Set `cron.sessionRetention: false` to disable.

### Troubleshooting

| Problem | Fix |
|---|---|
| "Nothing runs" | Check `openclaw gateway status`, ensure Gateway is running |
| Recurring job delays after failures | Check retry backoff; reset with `openclaw cron run <id>` |
| Telegram delivers to wrong place | Use explicit `:topic:` format for forum threads |
| Subagent announce retries | Set `delivery.bestEffort: true` |

---

## Webhooks

HTTP endpoints exposed by the Gateway for external triggers.

### Enable

```json5
{
  gateway: {
    webhooks: { enabled: true },
  },
}
```

### Auth

Webhooks require `gateway.auth.token` or `gateway.auth.password`. Pass via `Authorization: Bearer <token>` header.

### Endpoints

#### `POST /hooks/wake`

Trigger a wake/heartbeat cycle.

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake \
  -H "Authorization: Bearer $TOKEN"
```

#### `POST /hooks/agent`

Send a prompt to the agent, run a dedicated turn.

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message": "Check server status"}'
```

Options: `model`, `thinking`, `agentId`, `sessionKey`, `delivery`.

#### `POST /hooks/<name>` (mapped)

Custom-named webhook endpoints configured in `gateway.webhooks.map`.

### Session Key Policy

By default, each webhook call gets a fresh `hook:<uuid>` session key. Override with `sessionKey` in the request body.

### Responses

Returns `{ status: "ok", sessionId: "..." }` on success, or error details.

### Security

- Always use auth tokens
- Restrict network exposure (bind to loopback or use Tailscale)
- Webhooks inherit the agent's tool permissions

---

## Gmail Pub/Sub

Connect Gmail inbox notifications to OpenClaw via Google Cloud Pub/Sub.

### Prerequisites

- Google Cloud project with Gmail API + Pub/Sub enabled
- Service account or OAuth credentials
- A Pub/Sub topic and push subscription

### Setup

```bash
# Wizard (recommended)
openclaw webhooks gmail setup

# Manual one-time setup
# 1. Create GCP Pub/Sub topic
# 2. Grant Gmail publish rights
# 3. Configure OpenClaw
```

### Start the Watch

```bash
openclaw webhooks gmail watch
```

### Run the Push Handler

Configure the push subscription to POST to your Gateway's webhook endpoint.

### Troubleshooting

| Problem | Fix |
|---|---|
| No notifications | Verify Pub/Sub subscription is active |
| Auth errors | Check service account permissions |
| Webhook not receiving | Ensure Gateway is accessible from GCP |

### Cleanup

```bash
openclaw webhooks gmail stop
```

---

## Standing Orders

Grant agents permanent operating authority for defined programs. Define in `AGENTS.md`.

Each program requires:
- **Scope**: What the agent can do
- **Triggers**: When it executes
- **Approval gates**: What needs human approval
- **Escalation**: When to stop and ask for help

### Template

```markdown
## Program: [Name]
**Authority:** [What agent can do]
**Trigger:** [When it executes]
**Approval gate:** [What needs human approval]
**Escalation:** [When to stop and ask for help]
```

### Execution Discipline

Every task follows: Execute -> Verify -> Report.

---

---

## Background Tasks

Background tasks track detached agent runs (cron, webhooks, sub-agents, etc.).

> **Note**: ClawFlow (`openclaw flows`) is deprecated. Use `openclaw tasks` instead.

### CLI

```bash
openclaw tasks list                 # List tracked detached runs
openclaw tasks show <lookup>        # Show specific task details
openclaw tasks cancel <lookup>      # Cancel a running task
openclaw tasks audit                # Identify problematic task runs
```

### Migration from ClawFlow

| Old Command | New Command |
|---|---|
| `openclaw flows list` | `openclaw tasks list` |
| `openclaw flows show` | `openclaw tasks show` |
| `openclaw flows cancel` | `openclaw tasks cancel` |

---

## Hooks

Event-driven system for automating actions within the Gateway.

### Discovery Layers (Priority Order)

1. Bundled hooks
2. Plugin hooks
3. Managed hooks (`~/.openclaw/hooks/`)
4. Workspace hooks (`<workspace>/hooks/`, disabled by default)

Workspace hooks can introduce new hook names but cannot override hooks from higher-precedence sources.

### Bundled Hooks

| Hook | Description |
|---|---|
| `session-memory` | Preserves context during session resets |
| `bootstrap-extra-files` | Injects workspace files during init |
| `command-logger` | Records commands to `~/.openclaw/logs/commands.log` (JSONL) |
| `boot-md` | Executes BOOT.md at gateway start |

### Event Categories

- **Command**: `command:new`, `command:reset`, `command:stop`
- **Session**: `session:compact:before`, `session:compact:after`, `session:patch`
- **Agent**: `agent:bootstrap`
- **Gateway**: `gateway:startup`
- **Message**: `message:received`, `message:transcribed`, `message:preprocessed`, `message:sent`

### Hook Structure

```
my-hook/
├── HOOK.md          # Metadata (YAML frontmatter)
└── handler.ts       # Implementation
```

### Hook Packs (npm Distribution)

Hook packs are npm packages exporting multiple hooks via `openclaw.hooks` in `package.json`:

```bash
openclaw plugins install <hook-pack-spec>
```

### Plugin Hook API

Beyond event listeners, plugins can register sequential hooks:

| Hook | Purpose |
|---|---|
| `before_model_resolve` | Override model/provider before lookup |
| `before_prompt_build` | Modify system prompt |
| `before_tool_call` | Adjust parameters or block tool calls |
| `tool_result_persist` | Transform tool results before transcript storage |

28+ additional plugin hooks cover model I/O, lifecycle events, message flow, and subagent coordination.

### CLI Commands

```bash
openclaw hooks list                 # Show all discovered hooks
openclaw hooks enable <name>        # Activate a hook
openclaw hooks disable <name>       # Deactivate a hook
openclaw hooks info <name>          # Detailed hook info
openclaw hooks check                # Eligibility summary
```
