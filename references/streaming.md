# Streaming & Chunking

> Source: https://docs.openclaw.ai/concepts/streaming

OpenClaw implements two separate streaming layers:
- **Block streaming (channels)**: emits completed blocks as the assistant writes
- **Preview streaming (Telegram/Discord/Slack)**: updates temporary preview messages during generation

There is no true token-delta streaming to channel messages today.

## Block Streaming Architecture

```
Model output
  -> text_delta/events
       |-- (blockStreamingBreak=text_end)
       |    -> chunker emits blocks as buffer grows
       |-- (blockStreamingBreak=message_end)
            -> chunker flushes at message_end
                   -> channel send (block replies)
```

## Configuration

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off",       // "on" | "off"
      blockStreamingBreak: "text_end",    // "text_end" | "message_end"
      blockStreamingChunk: {
        minChars: 800,
        maxChars: 1200,
        breakPreference: "paragraph",     // paragraph > newline > sentence > whitespace
      },
      blockStreamingCoalesce: {
        idleMs: 1000,
        minChars: 1500,                   // Bumped for Signal/Slack/Discord
        maxChars: 4000,
      },
      humanDelay: {
        mode: "off",                      // "off" | "natural" (800-2500ms) | "custom"
        // minMs: 800,                    // For custom mode
        // maxMs: 2500,
      },
    },
  },
}
```

Channel-level overrides:
- `*.blockStreaming`: forces `"on"`/`"off"` per channel
- `*.textChunkLimit`: hard character cap per channel
- `*.chunkMode`: `length` (default) or `newline` (splits on blank lines)
- `channels.discord.maxLinesPerMessage`: default 17

**Critical**: Block streaming is off unless `*.blockStreaming` is explicitly `true`.

## Chunking Algorithm (EmbeddedBlockChunker)

- **Low bound**: don't emit until buffer >= `minChars` (unless forced)
- **High bound**: prefer splits before `maxChars`; if forced, split at `maxChars`
- **Break preference**: `paragraph` → `newline` → `sentence` → `whitespace` → hard break
- **Code fence handling**: Never split inside fences; when forced, close + reopen to keep Markdown valid
- **maxChars clamped** to channel `textChunkLimit`

## Boundary Semantics

| Break | Behavior |
|---|---|
| `text_end` | Stream blocks immediately as chunker emits; flush on each text_end event |
| `message_end` | Buffer all output until message completes, then flush |

## Coalescing (Merge Streamed Blocks)

Merges consecutive block chunks before sending:
- Waits for idle gaps (`idleMs`) before flushing
- Capped by `maxChars` (flushes if exceeded)
- Joiner from `breakPreference`: `paragraph` → `\n\n`, `newline` → `\n`, `sentence` → space

## Preview Streaming Modes

Config key: `channels.<channel>.streaming`

| Mode | Description |
|---|---|
| `off` | Disable preview streaming |
| `partial` | Single preview replaced with latest text |
| `block` | Preview updates in chunked/appended steps |
| `progress` | Status preview during generation, final answer at completion |

### Channel Support

| Channel | `off` | `partial` | `block` | `progress` |
|---|---|---|---|---|
| Telegram | yes | yes | yes | maps to `partial` |
| Discord | yes | yes | yes | maps to `partial` |
| Slack | yes | yes | yes | yes (native streaming) |

### Legacy Migration

- Telegram/Discord: `streamMode` + boolean `streaming` auto-migrate to `streaming` enum
- Slack: `streamMode` → `streaming` enum; boolean `streaming` → `nativeStreaming`

## Configuration Patterns

| Goal | Config |
|---|---|
| Stream chunks progressively | `blockStreamingDefault: "on"` + `blockStreamingBreak: "text_end"` + `*.blockStreaming: true` |
| Stream everything at end | `blockStreamingBreak: "message_end"` |
| Disable block streaming | `blockStreamingDefault: "off"` |
| Natural pacing | `humanDelay: { mode: "natural" }` |
