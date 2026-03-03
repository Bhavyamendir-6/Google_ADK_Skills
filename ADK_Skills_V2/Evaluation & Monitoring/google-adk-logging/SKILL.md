---
name: google-adk-logging
description: >
  Use this skill when a Google ADK agent needs structured logging of tool calls,
  agent decisions, session events, and errors. Covers configuring Python logging
  for ADK agents, writing structured log entries within tool functions and
  callbacks, integrating with Google Cloud Logging, and using ADK's built-in
  configurable logging functionality.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: evaluation-and-monitoring
---

## Purpose

This skill configures and implements structured logging for Google ADK agents.
ADK provides configurable logging functionality for monitoring and debugging
agent behavior. This skill covers setting up Python's `logging` module for ADK,
emitting structured log entries from tool functions and callbacks, capturing
invocation IDs and session IDs for log correlation, and shipping logs to
Google Cloud Logging for production observability.

## When to Use

- When debugging why an ADK agent produced an unexpected output or took a wrong
  tool call path.
- When setting up production monitoring for an ADK agent deployment.
- When integrating ADK agent logs with Google Cloud Logging or a centralized
  log aggregation system.
- When implementing audit logging for compliance (recording all tool calls,
  user inputs, and agent responses).
- When correlating logs across multiple tool calls within a single invocation.

## When NOT to Use

- Do not log raw user messages that contain PII without redaction.
- Do not use `print()` statements for production logging — use `logging` module.
- Do not log at `DEBUG` level in production — use `INFO` or `WARNING` to avoid
  log volume overflow.
- Do not rely solely on logging for observability — pair with tracing
  (`google-adk-tracing`) for distributed execution analysis.

## Google ADK Context

- **ADK configurable logging**: ADK uses Python's standard `logging` module.
  Configure it via `logging.basicConfig()` or `logging.config.dictConfig()`.
- **`tool_context.session_id`**: Session identifier available in all tool
  functions. Use as a correlation key in log entries.
- **`tool_context.user_id`**: User identifier for per-user log filtering.
- **`tool_context.invocation_id`** (via `_invocation_context`): Invocation
  identifier grouping all events in a single agent turn. Use for log correlation.
- **`CallbackContext`**: Available in before/after agent callbacks. Provides
  access to `session_id` and `user_id` for logging.
- **ADK Agent Execution Lifecycle**: Key logging points:
  - `before_agent_callback`: Log invocation start, user query, state snapshot.
  - Tool function entry/exit: Log tool name, args, result, and latency.
  - `after_agent_callback`: Log final response and state changes.
  - Error handlers: Log exception type, message, and stack trace.
- **Google Cloud Logging**: Use `google-cloud-logging` library to ship logs.
  ADK agents running on Cloud Run or GKE emit logs automatically to Cloud Logging.

## Capabilities

- Configures structured JSON logging for ADK agents.
- Logs tool call entry, exit, latency, and result in tool functions.
- Logs invocation lifecycle events in `before_agent_callback` and
  `after_agent_callback`.
- Attaches `session_id`, `user_id`, and `invocation_id` to every log entry.
- Ships logs to Google Cloud Logging for production observability.
- Implements audit logging for all tool calls and agent responses.
- Configures log sampling for high-throughput agents to manage log volume.

## Step-by-Step Instructions

1. **Configure logging** at application startup:
   ```python
   import logging
   import json

   logging.basicConfig(
       level=logging.INFO,
       format="%(asctime)s %(name)s %(levelname)s %(message)s",
   )
   logger = logging.getLogger("my_agent")
   ```

2. **Use structured log entries** (JSON format for Cloud Logging):
   ```python
   def log_structured(logger, level, message, **kwargs):
       entry = {"message": message, **kwargs}
       getattr(logger, level)(json.dumps(entry))
   ```

3. **Log tool calls** in tool functions with timing:
   ```python
   import time
   def my_tool(query: str, tool_context: ToolContext) -> dict:
       start = time.time()
       logger.info(json.dumps({
           "event": "tool_call_start",
           "tool": "my_tool",
           "session_id": tool_context.session_id,
           "user_id": tool_context.user_id,
           "args": {"query": query}
       }))
       result = perform_operation(query)
       latency_ms = (time.time() - start) * 1000
       logger.info(json.dumps({
           "event": "tool_call_end",
           "tool": "my_tool",
           "session_id": tool_context.session_id,
           "latency_ms": round(latency_ms, 2),
           "status": "success"
       }))
       return result
   ```

4. **Log invocation lifecycle** in callbacks:
   ```python
   def log_invocation_start(callback_context: CallbackContext) -> None:
       logger.info(json.dumps({
           "event": "invocation_start",
           "session_id": callback_context.session_id,
           "user_id": callback_context.user_id,
       }))
       return None
   ```

5. **Configure Google Cloud Logging** for production:
   ```python
   import google.cloud.logging
   client = google.cloud.logging.Client()
   client.setup_logging()
   ```

6. **Set log levels per environment**:
   - Development: `logging.DEBUG`
   - Staging: `logging.INFO`
   - Production: `logging.WARNING` (only warnings and errors)

7. **Add log correlation ID** by injecting `session_id` as a logging filter:
   ```python
   logging.getLogger().addFilter(SessionIDFilter(session_id))
   ```

## Input Format

```python
{
  "agent_id": "my_agent",
  "session_id": "sess_abc123",
  "tool_name": "search_products",
  "execution_time_ms": 342,
  "status": "success",
  "error": None,
  "metadata": {
    "log_level": "INFO",
    "environment": "production",
    "user_id": "user_xyz"
  }
}
```

## Output Format

```python
{
  "metric_recorded": True,
  "evaluation_result": "pass",
  "latency_ms": 342,
  "cost_estimate": 0.0,
  "issues_detected": [],
  "log_entries_emitted": 2,
  "log_destination": "google_cloud_logging"
}
```

## Error Handling

- **Logging backend unavailable** (Cloud Logging API down): Fall back to
  `StreamHandler` (stdout). Never let logging failures crash the agent.
  Wrap `logging` setup in try/except.
- **Missing `session_id`** in log context: Default to `"unknown_session"`.
  Log a warning that session correlation will be incomplete.
- **Log volume overflow**: Implement log sampling for high-frequency tools.
  Log every 10th call at `DEBUG`, but always log `ERROR` and `WARNING`.
- **PII in log messages**: Implement a `PIIRedactionFilter` that scrubs
  email, phone, and name patterns before emission.
- **Serialization error in JSON log entry**: Catch `TypeError` from
  `json.dumps()`. Fall back to `str()` representation.

## Examples

### Example 1 — Structured logging in a tool function

```python
import json
import time
import logging
from google.adk.tools import ToolContext

logger = logging.getLogger("my_agent.tools")

def search_products(query: str, max_results: int, tool_context: ToolContext) -> dict:
    """Searches the product catalog.

    Args:
        query (str): Search query string.
        max_results (int): Maximum number of results to return.
        tool_context (ToolContext): ADK tool context.

    Returns:
        dict: Contains 'results' (list) and 'total' (int).
    """
    start_ts = time.time()

    logger.info(json.dumps({
        "event": "tool_call_start",
        "tool": "search_products",
        "session_id": tool_context.session_id,
        "user_id": tool_context.user_id,
        "args": {"query": query, "max_results": max_results},
    }))

    try:
        results = _do_search(query, max_results)
        latency_ms = round((time.time() - start_ts) * 1000, 2)

        logger.info(json.dumps({
            "event": "tool_call_end",
            "tool": "search_products",
            "session_id": tool_context.session_id,
            "status": "success",
            "result_count": len(results),
            "latency_ms": latency_ms,
        }))

        return {"results": results, "total": len(results)}

    except Exception as exc:
        latency_ms = round((time.time() - start_ts) * 1000, 2)
        logger.error(json.dumps({
            "event": "tool_call_error",
            "tool": "search_products",
            "session_id": tool_context.session_id,
            "error": str(exc),
            "error_type": type(exc).__name__,
            "latency_ms": latency_ms,
        }))
        return {"error": str(exc), "error_type": "unexpected", "recoverable": False}

def _do_search(query, max_results):
    return []  # Placeholder
```

### Example 2 — Google Cloud Logging setup

```python
import google.cloud.logging
import logging

def setup_cloud_logging():
    """Configures Google Cloud Logging for production ADK agent."""
    client = google.cloud.logging.Client()
    client.setup_logging(log_level=logging.INFO)
    logger = logging.getLogger("my_agent")
    logger.info("Google Cloud Logging configured for ADK agent.")
    return logger
```

### Example 3 — Invocation lifecycle logging in callbacks

```python
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content
import json, logging, time

logger = logging.getLogger("my_agent.lifecycle")

_invocation_start_times: dict = {}

def log_before_agent(callback_context: CallbackContext) -> Optional[Content]:
    sid = callback_context.session_id
    _invocation_start_times[sid] = time.time()
    logger.info(json.dumps({
        "event": "invocation_start",
        "session_id": sid,
        "user_id": callback_context.user_id,
    }))
    return None

def log_after_agent(callback_context: CallbackContext) -> Optional[Content]:
    sid = callback_context.session_id
    start = _invocation_start_times.pop(sid, time.time())
    logger.info(json.dumps({
        "event": "invocation_end",
        "session_id": sid,
        "total_latency_ms": round((time.time() - start) * 1000, 2),
    }))
    return None
```

## Edge Cases

- **High-throughput agent** (100+ RPS): Log only `WARNING`/`ERROR` in production.
  Use log sampling at `INFO` level (1 in 10 calls).
- **Multi-agent (parallel)**: Each sub-agent has its own logger. Use shared
  `invocation_id` as a log correlation key across parallel sub-agents.
- **Failed tool call with no exception** (tool returns error dict): Log the
  error dict explicitly. The `logging` module only captures Python exceptions
  — not dict-based errors.
- **Very long log messages**: Truncate `args` and `result` fields to 500 chars
  in log entries. Log the full payload separately if needed.

## Integration Notes

- **`google-adk-observability`**: Logging is the foundation. Observability
  integrations (Phoenix, OpenTelemetry) consume the same structured log data.
- **`google-adk-tracing`**: Tracing provides distributed span-level details;
  logging provides human-readable event records. Use both together.
- **Google Cloud Logging**: Use `google-cloud-logging` library with `client.setup_logging()`.
  ADK agents on Cloud Run emit logs automatically.
- **`google-adk-error-analysis`**: Error log entries feed directly into the
  error analysis pipeline for root cause investigation.
