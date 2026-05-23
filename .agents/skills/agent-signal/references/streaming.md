# Streaming Signal Reference

This document describes how signals interact with streaming responses in the agent runtime.

## Overview

Streaming in LobeHub uses Server-Sent Events (SSE) to deliver incremental responses from LLM providers. Signals can interrupt, pause, or modify the stream at various points during processing.

## Stream Lifecycle

```
Request → Provider → Stream Start → Chunks → Stream End → Response
    ↑           ↑           ↑           ↑           ↑
  Signal      Signal      Signal      Signal      Signal
  (abort)    (timeout)   (start)    (chunk)     (complete)
```

## Signal Integration Points

### 1. Pre-Stream Signals

Fired before the stream begins. Can cancel the request entirely.

```typescript
// Signal: STREAM_BEFORE_START
{
  type: 'STREAM_BEFORE_START',
  payload: {
    requestId: string;
    model: string;
    provider: string;
    messages: ChatMessage[];
  }
}
```

### 2. Stream Chunk Signals

Fired for each chunk received from the provider.

```typescript
// Signal: STREAM_CHUNK_RECEIVED
{
  type: 'STREAM_CHUNK_RECEIVED',
  payload: {
    requestId: string;
    chunkIndex: number;
    delta: string;
    finishReason: string | null;
    usage?: TokenUsage;
  }
}
```

### 3. Stream Abort Signals

Fired when the stream is aborted by the user or system.

```typescript
// Signal: STREAM_ABORTED
{
  type: 'STREAM_ABORTED',
  payload: {
    requestId: string;
    reason: 'user_cancel' | 'timeout' | 'error' | 'system';
    chunksReceived: number;
    partialContent: string;
  }
}
```

### 4. Stream Complete Signals

Fired when the stream ends successfully.

```typescript
// Signal: STREAM_COMPLETE
{
  type: 'STREAM_COMPLETE',
  payload: {
    requestId: string;
    totalChunks: number;
    finalContent: string;
    usage: TokenUsage;
    duration: number; // ms
  }
}
```

## AbortController Integration

Signals connect to the native `AbortController` API to halt streams:

```typescript
const controller = new AbortController();

// Register signal handler
onSignal('STREAM_ABORT_REQUEST', (signal) => {
  controller.abort(signal.payload.reason);
});

// Pass to fetch
fetch(providerUrl, {
  signal: controller.signal,
  // ...
});
```

## Backpressure Handling

When downstream consumers are slow, signals help manage backpressure:

| Signal | Trigger | Action |
|--------|---------|--------|
| `STREAM_BUFFER_FULL` | Buffer exceeds threshold | Pause upstream reads |
| `STREAM_BUFFER_DRAINED` | Buffer below threshold | Resume upstream reads |
| `STREAM_CONSUMER_SLOW` | Chunk processing > 500ms | Emit warning signal |

## Error Recovery

Streaming errors emit specific signals that can trigger retry logic:

```
STREAM_ERROR
  ├── STREAM_RETRY_ATTEMPT  (retryable errors)
  └── STREAM_FATAL_ERROR    (non-retryable errors)
```

## Provider-Specific Notes

- **OpenAI**: Uses `[DONE]` sentinel; signal fired on receipt
- **Anthropic**: Uses `event: message_stop`; maps to `STREAM_COMPLETE`
- **Ollama**: Local stream; no network timeout signals
- **Azure OpenAI**: Same as OpenAI with additional deployment signals

## Related

- [Signal Types](./signal-types.md)
- [Signal Flow](./signal-flow.md)
- [Error Handling](./error-handling.md)
- [Provider Integration](./provider-integration.md)
