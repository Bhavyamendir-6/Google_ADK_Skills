---
name: tool-retries
description: >
  Use this skill when a recoverable tool failure should trigger an automatic
  retry with exponential backoff. Covers retry strategy design, backoff
  calculation, jitter, maximum attempt limits, and abort conditions for
  agent tool calls.
compatibility:
  - Claude Code
  - Google ADK
  - OpenAI Agents
  - MCP-based agents
  - Tool-using LLM agents
metadata:
  author: agent-skills-generator
  version: "1.0"
---

## Purpose

This skill implements retry logic for tool calls that fail due to transient
errors (network timeouts, rate limits, temporary service outages). It ensures
the agent does not give up prematurely on recoverable failures, while also
preventing infinite loops through maximum attempt caps and abort conditions.

## When to Use

- When a tool returns `{"recoverable": True}` from error handling.
- When calling external APIs subject to rate limits or intermittent failures.
- When implementing production-grade tool runners that must tolerate flaky
  services.

## When NOT to Use

- Do not retry on `permanent` or `invalid_input` errors — retrying will not fix them.
- Do not retry when `recoverable` is `False`.
- Do not retry when the tool has side effects that must not be duplicated
  (e.g., sending an email, writing to a database without idempotency).

## Capabilities

- Implements exponential backoff with configurable base delay and multiplier.
- Adds random jitter to prevent thundering-herd retries.
- Caps total retry attempts at a configurable maximum.
- Skips retry for non-recoverable errors.
- Returns the last error result after all attempts are exhausted.
- Logs each retry attempt with delay and attempt number.

## Step-by-Step Instructions

1. **Check `recoverable` field** in the error result before retrying:
   ```python
   if not result.get("recoverable", False):
       return result  # Do not retry
   ```

2. **Configure retry parameters**:
   - `max_attempts`: 3 (default). Increase to 5 for high-priority external APIs.
   - `base_delay_seconds`: 1.0
   - `backoff_multiplier`: 2.0
   - `jitter_factor`: 0.2 (±20% random variance)

3. **Implement the retry loop**:
   ```
   for attempt in 1..max_attempts:
       result = call_tool(...)
       if success: return result
       if not recoverable: return result
       delay = base_delay * (backoff_multiplier ^ (attempt - 1))
       jitter = random(-jitter_factor, +jitter_factor) * delay
       sleep(delay + jitter)
   return last_error_result
   ```

4. **Log each retry** at WARNING level:
   ```
   WARNING: [tool_name] Attempt 2/3 failed. Retrying in 2.3s. Error: <msg>
   ```

5. **After max attempts**, return the last error result with an additional
   `"retry_exhausted": True` key.

6. **Wrap the retry logic in a decorator** for reuse across all tools.

## Input Format

```
tool_fn: Callable         # The tool function to retry
tool_args: dict           # Arguments to pass to the tool
max_attempts: int         # Default: 3
base_delay: float         # Default: 1.0 seconds
backoff_multiplier: float # Default: 2.0
jitter_factor: float      # Default: 0.2
```

## Output Format

On success (any attempt):
```python
{"result": ..., "attempt": 2}  # Successful result with attempt number
```

On exhaustion:
```python
{
  "error": "Tool call failed after 3 attempts.",
  "error_type": "transient",
  "recoverable": False,
  "retry_exhausted": True,
  "last_detail": "ConnectionError: ..."
}
```

## Error Handling

- **`recoverable: False`**: Return immediately, skip all retry logic.
- **Exception raised despite error handler**: This should not happen if
  `tool-error-handling` is applied. If it does, treat as non-recoverable.
- **Max attempts reached**: Set `recoverable: False` in the returned dict
  to prevent upstream callers from retrying again.

## Examples

### Example 1 — Retry decorator with exponential backoff

```python
import time
import random
import logging
import functools

logger = logging.getLogger(__name__)

def with_retries(max_attempts: int = 3, base_delay: float = 1.0,
                 backoff_multiplier: float = 2.0, jitter_factor: float = 0.2):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            last_result = None
            for attempt in range(1, max_attempts + 1):
                result = fn(*args, **kwargs)
                if "error" not in result:
                    return result
                if not result.get("recoverable", False):
                    return result
                last_result = result
                if attempt < max_attempts:
                    delay = base_delay * (backoff_multiplier ** (attempt - 1))
                    jitter = random.uniform(-jitter_factor, jitter_factor) * delay
                    sleep_time = max(0.0, delay + jitter)
                    logger.warning(
                        f"[{fn.__name__}] Attempt {attempt}/{max_attempts} failed. "
                        f"Retrying in {sleep_time:.2f}s. Error: {result.get('error')}"
                    )
                    time.sleep(sleep_time)
            last_result["retry_exhausted"] = True
            last_result["recoverable"] = False
            last_result["error"] = f"Tool call failed after {max_attempts} attempts."
            return last_result
        return wrapper
    return decorator

@with_retries(max_attempts=3, base_delay=1.0)
@tool_error_handler
def get_weather(city: str) -> dict:
    """Retrieves current weather for a city."""
    ...
```

### Example 2 — Inline retry within agent loop

```python
for attempt in range(1, 4):
    result = call_tool(tool_fn, tool_args)
    if "error" not in result or not result.get("recoverable"):
        break
    time.sleep(2 ** (attempt - 1))
```

## Edge Cases

- **Idempotency required**: Only retry tools that are idempotent (read-only or
  idempotent writes). Add `idempotent: True` metadata to safe tools.
- **Rate limit with Retry-After header**: Parse the `Retry-After` value from
  the HTTP response and use it as the sleep duration instead of backoff.
- **Nested retries (tool uses a retry internally)**: Set `max_attempts=1` at
  the outer layer to avoid compounding retry counts.
- **Async tools**: Use `asyncio.sleep` instead of `time.sleep` in async
  contexts.

## Integration Notes

- **Google ADK**: Apply `@with_retries` decorator on top of `@tool_error_handler`
  on every tool function that calls an external service.
- **OpenAI Agents SDK**: The SDK does not implement retries natively; apply
  this decorator at the function level.
- **MCP**: MCP tool servers should implement retries server-side. Apply this
  skill when building the MCP server's tool handlers.
- **tool-error-handling skill**: This skill depends on `tool-error-handling`
  to produce the `recoverable` field that drives retry decisions.
