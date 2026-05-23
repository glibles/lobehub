# Provider Integration with Agent Signals

This document describes how AI providers integrate with the agent signal system to emit and handle signals during inference.

## Overview

Each provider in the agent runtime implements a signal-aware interface that allows it to:

1. Emit signals at key lifecycle points (start, token, tool call, end, error)
2. Respect abort signals from the consumer
3. Report observability data through signal hooks

## Provider Signal Contract

Every provider must implement the `SignalAwareProvider` interface:

```typescript
interface SignalAwareProvider {
  /**
   * The AbortSignal passed from the client to cancel the stream.
   */
  abortSignal?: AbortSignal;

  /**
   * Emit a signal event to registered listeners.
   */
  emit(signal: AgentSignal): void;

  /**
   * Called when the provider stream begins.
   */
  onStreamStart?(meta: StreamStartMeta): void;

  /**
   * Called for each token chunk received from the model.
   */
  onToken?(chunk: TokenChunk): void;

  /**
   * Called when a tool/function call is detected in the stream.
   */
  onToolCall?(call: ToolCallSignal): void;

  /**
   * Called when the stream completes successfully.
   */
  onStreamEnd?(summary: StreamEndSummary): void;

  /**
   * Called when an error occurs during streaming.
   */
  onError?(error: StreamError): void;
}
```

## OpenAI Provider Integration

The OpenAI provider emits signals via the streaming response iterator:

```typescript
async function* streamOpenAI(params, options) {
  const stream = await openai.chat.completions.create({
    ...params,
    stream: true,
  }, { signal: options.abortSignal });

  options.emit({ type: 'stream_start', provider: 'openai', timestamp: Date.now() });

  for await (const chunk of stream) {
    const delta = chunk.choices[0]?.delta;

    if (delta?.content) {
      options.emit({ type: 'token', content: delta.content });
      yield delta.content;
    }

    if (delta?.tool_calls) {
      options.emit({ type: 'tool_call', calls: delta.tool_calls });
    }
  }

  options.emit({ type: 'stream_end', provider: 'openai', timestamp: Date.now() });
}
```

## Abort Signal Propagation

The abort signal flows from the HTTP request through to the provider:

```
HTTP Request
  └─> AgentRuntime.chat()
        └─> Provider.createStream()
              └─> fetch() / SDK call
                    └─> AbortController.signal
```

When the client disconnects or explicitly cancels:

1. The `AbortController` fires
2. The provider's fetch/SDK call throws `AbortError`
3. The provider emits `{ type: 'aborted' }` signal
4. All registered hooks receive the abort signal
5. Resources are cleaned up

## Adding Signal Support to a New Provider

To add signal support to a new provider:

1. Accept `SignalOptions` in the provider's stream method
2. Wrap the streaming loop with signal emissions
3. Pass `abortSignal` to the underlying SDK or fetch call
4. Handle `AbortError` and emit the `aborted` signal
5. Register the provider in `agent-signal/agents/`

See `agents/openai.yaml` for a reference configuration.

## Signal Batching

For high-frequency token signals, providers may batch emissions to reduce overhead:

```typescript
// Emit at most every 16ms (roughly 60fps)
const batchedEmit = throttle(options.emit, 16);
```

Note: `stream_start`, `stream_end`, `tool_call`, `error`, and `aborted` signals are **never** batched.
