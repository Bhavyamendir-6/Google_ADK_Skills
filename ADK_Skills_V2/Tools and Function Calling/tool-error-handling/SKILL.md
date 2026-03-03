---
name: tool-error-handling
description: >
  Use this skill when a tool call fails, raises an exception, or returns an
  error response. Covers catching errors, classifying failure types, returning
  structured error results to the LLM, and deciding whether to retry or abort.
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

This skill governs how agent tools detect, classify, and communicate errors
back to the LLM without crashing the agent loop. It ensures that tool failures
are surfaced as structured data that the LLM can reason about, rather than as
unhandled exceptions that terminate the session.

## When to Use

- When a tool call may raise exceptions (network errors, auth failures, invalid
  data).
- When implementing the try/except wrapper around tool execution in the
  agent loop.
- When a tool returns a partial or empty result that counts as a logical error.
- When deciding whether to retry, fall back, or abort after a tool failure.

## When NOT to Use

- Do not use this skill to implement retry logic (see `tool-retries`).
- Do not use to validate input arguments before the call (see `tool-validation`).
- Do not swallow errors silently — always return structured error output.

## Capabilities

- Wraps tool execution in a standardized error handling decorator.
- Classifies errors into: `transient`, `permanent`, `invalid_input`, `not_found`.
- Returns structured `{"error": ..., "error_type": ..., "recoverable": bool}`.
- Prevents unhandled exceptions from breaking the agent loop.
- Preserves the original exception message for debugging.

## Step-by-Step Instructions

1. **Wrap every tool execution** in a try/except block. Never let exceptions
   propagate out of a tool call into the agent runner.

2. **Classify the exception type**:
   - `transient`: Network timeouts, rate limits, temporary service unavailability.
   - `permanent`: Auth errors, resource not found, schema violations.
   - `invalid_input`: LLM passed wrong argument type or value.
   - `unexpected`: All other exceptions.

3. **Map exception classes to error types**:
   ```python
   TRANSIENT_ERRORS = (TimeoutError, ConnectionError, RateLimitError)
   PERMANENT_ERRORS = (AuthenticationError, NotFoundError, PermissionError)
   INVALID_INPUT_ERRORS = (ValueError, TypeError, KeyError)
   ```

4. **Return a structured error dict** as the tool result:
   ```python
   {
     "error": "Human-readable description of what went wrong",
     "error_type": "transient" | "permanent" | "invalid_input" | "unexpected",
     "recoverable": True | False,
     "detail": str(original_exception)
   }
   ```

5. **Log the error** with the tool name, input args, error type, and traceback
   at WARNING or ERROR level.

6. **Do NOT re-raise** the exception. The agent loop must continue.

7. **Include `"recoverable": True`** only for transient errors where a retry
   is safe. The `tool-retries` skill reads this field.

## Input Format

```
tool_name: <str>
tool_args: dict
exception: <Exception instance> | None
logical_error: dict | None  # e.g., empty API response
```

## Output Format

Error result dict (returned as tool call result to the LLM):

```python
{
  "error": "The weather API is temporarily unavailable. Please try again.",
  "error_type": "transient",
  "recoverable": True,
  "detail": "ConnectionError: ('Connection aborted.', RemoteDisconnected(...))"
}
```

## Error Handling

- **KeyError when accessing result fields**: Classify as `invalid_input` if
  args were wrong, else `unexpected`.
- **HTTP 4xx errors**: `401/403` → `permanent`. `429` → `transient`.
  `400` → `invalid_input`. `404` → `not_found` (subtype of `permanent`).
- **HTTP 5xx errors**: Always `transient`.
- **JSON decode errors on API response**: `unexpected` — the API contract changed.

## Examples

### Example 1 — Standardized error handler decorator

```python
import functools
import logging
import traceback

logger = logging.getLogger(__name__)

TRANSIENT = (TimeoutError, ConnectionError)
PERMANENT = (PermissionError, FileNotFoundError)
INVALID_INPUT = (ValueError, TypeError)

def tool_error_handler(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        try:
            return fn(*args, **kwargs)
        except TRANSIENT as e:
            logger.warning(f"[{fn.__name__}] Transient error: {e}")
            return {"error": str(e), "error_type": "transient", "recoverable": True, "detail": traceback.format_exc()}
        except PERMANENT as e:
            logger.error(f"[{fn.__name__}] Permanent error: {e}")
            return {"error": str(e), "error_type": "permanent", "recoverable": False, "detail": traceback.format_exc()}
        except INVALID_INPUT as e:
            logger.warning(f"[{fn.__name__}] Invalid input: {e}")
            return {"error": str(e), "error_type": "invalid_input", "recoverable": False, "detail": traceback.format_exc()}
        except Exception as e:
            logger.error(f"[{fn.__name__}] Unexpected error: {e}")
            return {"error": "An unexpected error occurred.", "error_type": "unexpected", "recoverable": False, "detail": traceback.format_exc()}
    return wrapper

@tool_error_handler
def get_weather(city: str) -> dict:
    """Retrieves weather data for a given city."""
    ...
```

### Example 2 — Logical error (empty result)

```python
def search_products(query: str) -> dict:
    """Searches product catalog for a query term."""
    results = product_db.search(query)
    if not results:
        return {
            "error": f"No products found for query '{query}'.",
            "error_type": "not_found",
            "recoverable": False,
        }
    return {"results": results, "count": len(results)}
```

## Edge Cases

- **Tool raises SystemExit or KeyboardInterrupt**: Do NOT catch these — let
  them propagate. Only catch `Exception` subclasses.
- **Nested tool calls**: Each tool handles its own errors. Do not double-wrap.
- **Partial success**: If a tool partially succeeds (e.g., 3 of 5 items
  processed), return both a `"partial_result"` and an `"error"` key.
- **Error message too long**: Truncate `"detail"` to 500 characters before
  returning to the LLM context.

## Integration Notes

- **Google ADK**: Unhandled exceptions in tool functions crash the ADK runner.
  Always apply the `@tool_error_handler` decorator.
- **OpenAI Agents SDK**: The SDK catches tool exceptions internally but returns
  a generic error string. Use explicit error dicts for better LLM reasoning.
- **tool-retries skill**: Reads `recoverable: True` to determine if a retry
  is safe. Do not retry when `recoverable: False`.
- **tool-chaining skill**: Checks for `"error"` key in outputs to abort chains
  on failure. This skill produces that key consistently.
