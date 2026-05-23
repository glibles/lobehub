# Agent Signal Error Handling

This document describes error handling patterns for the agent signal system.

## Overview

The agent signal system uses a structured error hierarchy to provide consistent error handling across all providers and signal types.

## Error Types

### SignalError

Base error class for all signal-related errors.

```typescript
export class SignalError extends Error {
  constructor(
    message: string,
    public readonly code: SignalErrorCode,
    public readonly signal?: AgentSignal,
    public readonly cause?: Error
  ) {
    super(message);
    this.name = 'SignalError';
  }
}
```

### SignalErrorCode

```typescript
export enum SignalErrorCode {
  // Signal lifecycle errors
  SIGNAL_NOT_FOUND = 'SIGNAL_NOT_FOUND',
  SIGNAL_ALREADY_ABORTED = 'SIGNAL_ALREADY_ABORTED',
  SIGNAL_TIMEOUT = 'SIGNAL_TIMEOUT',

  // Provider errors
  PROVIDER_UNAVAILABLE = 'PROVIDER_UNAVAILABLE',
  PROVIDER_RATE_LIMITED = 'PROVIDER_RATE_LIMITED',
  PROVIDER_AUTH_FAILED = 'PROVIDER_AUTH_FAILED',

  // Stream errors
  STREAM_INTERRUPTED = 'STREAM_INTERRUPTED',
  STREAM_PARSE_ERROR = 'STREAM_PARSE_ERROR',

  // Hook errors
  HOOK_EXECUTION_FAILED = 'HOOK_EXECUTION_FAILED',
}
```

## Error Propagation

### In Providers

Providers should catch native errors and wrap them in `SignalError`:

```typescript
async function callProvider(signal: AgentSignal) {
  try {
    const response = await fetch(endpoint, {
      signal: signal.abortSignal,
    });
    return response;
  } catch (err) {
    if (err instanceof DOMException && err.name === 'AbortError') {
      throw new SignalError(
        'Request aborted by signal',
        SignalErrorCode.SIGNAL_ALREADY_ABORTED,
        signal,
        err
      );
    }
    throw new SignalError(
      'Provider request failed',
      SignalErrorCode.PROVIDER_UNAVAILABLE,
      signal,
      err as Error
    );
  }
}
```

### In Hooks

Hook errors are isolated to prevent cascading failures:

```typescript
async function runHookSafely<T>(
  hook: SignalHook<T>,
  context: SignalHookContext<T>
): Promise<void> {
  try {
    await hook(context);
  } catch (err) {
    console.error(
      `[AgentSignal] Hook '${hook.name}' failed:`,
      err
    );
    // Hook errors do not abort the signal by default
  }
}
```

## Retry Strategy

The signal system supports configurable retry behavior:

```typescript
interface RetryConfig {
  maxAttempts: number;
  backoffMs: number;
  retryOn: SignalErrorCode[];
}

const DEFAULT_RETRY_CONFIG: RetryConfig = {
  maxAttempts: 3,
  backoffMs: 500,
  retryOn: [
    SignalErrorCode.PROVIDER_RATE_LIMITED,
    SignalErrorCode.STREAM_INTERRUPTED,
  ],
};
```

## Observability

All errors are automatically reported to the observability layer (see `observability.md`):

- Error code and message
- Associated signal ID
- Stack trace (in development)
- Provider context

## Best Practices

1. **Always wrap provider errors** — Never let raw fetch/axios errors escape a provider.
2. **Use error codes** — Prefer `SignalErrorCode` over string matching for error handling logic.
3. **Preserve cause chains** — Pass the original error as `cause` to maintain stack traces.
4. **Don't swallow hook errors** — Log them but allow the signal to continue.
5. **Abort on unrecoverable errors** — Call `signal.abort()` when the error is terminal.
