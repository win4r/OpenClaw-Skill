# Command Queue

> Source: https://docs.openclaw.ai/concepts/queue

## Why

Auto-reply runs can be expensive (LLM calls) and collide when multiple inbound messages arrive simultaneously. Serialization prevents competing for shared resources (session files, logs, CLI stdin) and reduces upstream rate limits.

## How It Works

- Lane-aware FIFO queue drains each lane with configurable concurrency cap
- Default concurrency: unconfigured lanes = 1, `main` = 4, `subagent` = 8
- `runEmbeddedPiAgent` enqueues by session key (lane `session:<key>`) — only one active run per session
- Each session run queues into global lane (`main`) — capped by `agents.defaults.maxConcurrent`
- Typing indicators fire immediately on enqueue (when channel-supported)
- Additional lanes (`cron`, `subagent`) allow background jobs to execute without blocking inbound replies

## Queue Modes (Per Channel)

| Mode | Behavior |
|---|---|
| `steer` | Inject immediately into current run; cancels pending tool calls after next tool boundary; falls back to followup if not streaming |
| `followup` | Enqueue for next agent turn after current run ends |
| `collect` | Coalesce all queued messages into single followup turn (default); messages targeting different channels/threads drain individually |
| `steer-backlog` | Steer now and preserve message for followup turn |
| `interrupt` (legacy) | Abort active session run; execute newest message |
| `queue` (legacy alias) | Same as `steer` |

## Queue Options

| Option | Description | Default |
|---|---|---|
| `debounceMs` | Wait for quiet before starting followup turn | `1000` |
| `cap` | Max queued messages per session | `20` |
| `drop` | Overflow policy: `old`, `new`, `summarize` | `summarize` |

`summarize` keeps a short bullet list of dropped messages and injects it as a synthetic followup prompt.

## Configuration

```json5
{
  messages: {
    queue: {
      mode: "collect",
      debounceMs: 1000,
      cap: 20,
      drop: "summarize",
      byChannel: {
        discord: "collect",
        whatsapp: "collect",
      },
    },
    inbound: {
      debounceMs: 2000,          // Inbound debounce (separate from queue)
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

## Per-Session Overrides

```
/queue <mode>                              # Set mode for current session
/queue collect debounce:2s cap:25 drop:summarize  # Combine options
/queue default                             # Clear override
/queue reset                               # Clear override
```

## Scope & Guarantees

- Applies to auto-reply agent runs across all inbound channels
- Default lane (`main`) is process-wide for inbound + main heartbeats
- Per-session lanes guarantee only one agent run per session simultaneously
- Pure TypeScript + promises — no external dependencies or worker threads

## Troubleshooting

- Enable verbose logs; search for `"queued for ...ms"` lines
- If runs pile up, check `agents.defaults.maxConcurrent` setting
- For high-volume channels, consider `collect` mode with higher `debounceMs`
