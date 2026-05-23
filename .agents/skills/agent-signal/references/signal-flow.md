# Agent Signal Flow

This document describes the end-to-end flow of signals through the agent runtime system.

## Overview

Signals are the primary mechanism for communicating state changes, errors, and lifecycle events between the agent runtime and its consumers. The flow follows a unidirectional data pattern.

```
User Request
    │
    ▼
┌─────────────────┐
│  Agent Runtime  │
│  (Entry Point)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌──────────────────┐
│  Signal Emitter │────▶│  Signal Registry  │
└─────────────────┘     └────────┬─────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
     ┌────────────────┐  ┌───────────────┐  ┌────────────────┐
     │  Hook Handlers │  │  Observability│  │  Error Handler │
     └────────────────┘  └───────────────┘  └────────────────┘
```

## Signal Lifecycle

### 1. Signal Creation

Signals are created at key points in the agent execution pipeline:

- **`onStart`** — Emitted when the agent begins processing a request
- **`onChunk`** — Emitted for each streamed token or data chunk
- **`onToolCall`** — Emitted when the agent invokes a tool
- **`onToolResult`** — Emitted when a tool returns a result
- **`onFinish`** — Emitted when the agent completes successfully
- **`onError`** — Emitted when an unrecoverable error occurs
- **`onAbort`** — Emitted when the request is cancelled by the client

### 2. Signal Propagation

Each signal propagates through a chain of registered handlers in priority order:

```
Signal Emitted
    │
    ▼
Pre-processing hooks (priority: 0–49)
    │
    ▼
Core handlers (priority: 50–99)
    │
    ▼
Post-processing hooks (priority: 100+)
    │
    ▼
Observability sinks (async, non-blocking)
```

### 3. Signal Cancellation

Any handler in the chain may call `signal.stopPropagation()` to prevent downstream handlers from receiving the signal. This is useful for:

- Short-circuiting on error conditions
- Implementing feature flags that suppress certain behaviors
- Rate limiting at the handler level

## Provider-Specific Signal Mapping

Different LLM providers emit signals at slightly different points. The runtime normalizes these into a consistent interface:

| Provider  | Stream Event       | Normalized Signal |
|-----------|--------------------|-------------------|
| OpenAI    | `data: [DONE]`     | `onFinish`        |
| Anthropic | `message_stop`     | `onFinish`        |
| Google    | `finishReason`     | `onFinish`        |
| OpenAI    | `tool_calls`       | `onToolCall`      |
| Anthropic | `tool_use`         | `onToolCall`      |

## Error Signal Flow

When an error occurs, the signal system guarantees:

1. The `onError` signal is always emitted before the stream closes
2. Any in-flight `onChunk` signals are drained before `onError`
3. `onFinish` is **not** emitted if `onError` fires
4. The original error is attached to the signal payload as `signal.error`

```typescript
// Example error signal payload
{
  type: 'onError',
  timestamp: 1710000000000,
  requestId: 'req_abc123',
  error: {
    code: 'RATE_LIMIT_EXCEEDED',
    message: 'Too many requests',
    provider: 'openai',
    retryAfter: 30,
  }
}
```

## Abort Signal Integration

The agent signal system integrates with the Web `AbortController` API:

```typescript
const controller = new AbortController();

// Aborting the controller triggers the onAbort signal
controller.abort();

// The signal system maps AbortError to the onAbort signal type
// rather than routing it through onError
```

This distinction is important for observability — aborted requests should not
count as errors in SLO calculations.

## Backpressure Handling

For high-throughput scenarios, the signal system implements backpressure via:

- **Buffered emission**: `onChunk` signals are buffered in a ring buffer (default size: 64)
- **Async handlers**: Handlers that return a `Promise` are awaited before the next signal is dispatched
- **Overflow policy**: When the buffer is full, the oldest unprocessed signal is dropped and a `onBufferOverflow` diagnostic signal is emitted

## Related Documents

- [Signal Types](./signal-types.md) — Full reference for all signal shapes
- [Handlers](./handlers.md) — How to register and implement signal handlers
- [Observability](./observability.md) — Connecting signals to telemetry pipelines
- [Architecture](./architecture.md) — High-level system design
