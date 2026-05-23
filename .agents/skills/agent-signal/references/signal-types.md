# Agent Signal Types Reference

This document describes the signal types used in the agent runtime system for controlling and monitoring agent execution.

## Overview

Signals provide a mechanism for external systems to communicate with running agents, enabling cancellation, pausing, and priority adjustments during execution.

## Core Signal Types

### AbortSignal

Used to cancel an ongoing agent operation. Follows the Web API `AbortSignal` standard.

```typescript
interface AgentAbortSignal {
  type: 'abort';
  reason?: string;
  timestamp: number;
}
```

**Usage:**
- User-initiated cancellation
- Timeout enforcement
- Resource cleanup on navigation

**Example:**
```typescript
const controller = new AbortController();
const { signal } = controller;

// Pass to agent runtime
await agentRuntime.chat(params, { signal });

// Cancel from outside
controller.abort('User cancelled the request');
```

---

### PauseSignal

Allows temporarily suspending agent execution without full cancellation.

```typescript
interface AgentPauseSignal {
  type: 'pause';
  resumeToken?: string;
  maxPauseDuration?: number; // milliseconds
}
```

**Usage:**
- Rate limiting compliance
- User-requested pause
- System resource management

---

### ThrottleSignal

Controls the rate at which tokens are streamed to the client.

```typescript
interface AgentThrottleSignal {
  type: 'throttle';
  tokensPerSecond: number;
  burstLimit?: number;
}
```

---

## Signal Priority Levels

| Priority | Value | Description |
|----------|-------|-------------|
| `CRITICAL` | 0 | Immediate abort, no cleanup |
| `HIGH` | 1 | Graceful abort with cleanup |
| `NORMAL` | 2 | Standard signal handling |
| `LOW` | 3 | Best-effort, may be deferred |

## Signal Lifecycle

```
CREATED → PENDING → DISPATCHED → HANDLED
                              ↘ IGNORED
                              ↘ FAILED
```

### States

- **CREATED**: Signal instantiated but not yet sent
- **PENDING**: Signal queued for dispatch
- **DISPATCHED**: Signal sent to the agent runtime
- **HANDLED**: Agent acknowledged and acted on signal
- **IGNORED**: Agent was in a state where signal could not be applied
- **FAILED**: Signal handling encountered an error

## Signal Context

Signals carry contextual metadata to assist with observability and debugging:

```typescript
interface SignalContext {
  traceId: string;        // Correlates with the originating request
  userId?: string;        // User who triggered the signal
  sessionId: string;      // Agent session identifier
  source: SignalSource;   // Origin of the signal
  metadata?: Record<string, unknown>;
}

type SignalSource =
  | 'user_action'       // Direct user interaction
  | 'timeout'           // Automatic timeout
  | 'rate_limit'        // Provider rate limiting
  | 'system'            // Internal system signal
  | 'webhook';          // External webhook trigger
```

## Integration with OpenAI Provider

The OpenAI provider handles signals through the `AbortController` pattern:

```typescript
// See: agents/openai.yaml for provider configuration
// See: handlers.md for implementation details

const handleSignal = (signal: AgentAbortSignal, controller: AbortController) => {
  if (signal.type === 'abort') {
    controller.abort(signal.reason);
  }
};
```

## Related Documents

- [Architecture Overview](./architecture.md)
- [Signal Handlers](./handlers.md)
- [Observability](./observability.md)
