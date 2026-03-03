---
name: google-adk-metadata-handling
description: >
  Use this skill when a Google ADK agent needs to attach, read, or manage
  non-state metadata on sessions or invocations — such as request identifiers,
  client version, IP address, tracing headers, or feature flags. Covers
  storing metadata in tool_context.metadata or session-scoped state keys,
  and using metadata for routing, observability, and audit logging.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: state-and-context-handling
---

## Purpose

This skill governs how supplementary, non-conversational metadata is attached
to ADK sessions and invocations. Metadata includes request-level attributes
(tracing headers, API version, client platform) and session-level attributes
(environment flags, A/B test variant, routing rules) that inform agent behavior
or observability systems without being part of the conversation state itself.

## When to Use

- When you need to pass request-level metadata (trace ID, client version) into
  the ADK tool execution context.
- When an agent's behavior must vary based on deployment environment or feature
  flag metadata (e.g., `env: staging`, `feature: beta_search`).
- When implementing audit logging that requires attaching request metadata to
  each invocation.
- When routing multi-agent calls based on metadata attributes.

## When NOT to Use

- Do not use metadata for conversation state or user preferences — use
  `session.state` with appropriate prefixes.
- Do not store sensitive PII in metadata — use encrypted state or external
  secure storage.
- Do not use metadata as a replacement for proper session state management.

## Google ADK Context

- **`session.state` with `app:` prefix**: App-scoped state serves as the primary
  mechanism for global metadata accessible across all sessions. E.g.,
  `app:api_version`, `app:feature_flags`.
- **`tool_context.metadata`**: ADK provides `tool_context` in tool functions.
  While `tool_context` does not have a formal `.metadata` attribute in the public
  API, metadata can be passed via `session.state["meta:key"]` pattern using a
  custom `meta:` naming convention.
- **`tool_context.session_id`** and **`tool_context.user_id`**: Available in all
  tool functions for constructing trace identifiers.
- **`InvocationContext`**: The internal context object accessible via
  `callback_context._invocation_context` in callbacks. Contains the full session
  and invocation ID.
- **ADK Execution Lifecycle**: Metadata should be attached at session creation
  or at the start of an invocation in `before_agent_callback`.
- **Observability**: ADK emits events for each invocation. Attach trace IDs to
  events for correlation in Cloud Trace or custom logging systems.

## Capabilities

- Attaches request metadata (trace IDs, client version) to sessions at creation.
- Reads invocation metadata in tool functions via `session.state["meta:key"]`.
- Propagates trace headers through tool calls for distributed tracing.
- Implements feature flag resolution from `app:` or `meta:` state.
- Attaches routing metadata used by `SequentialAgent` or `ParallelAgent`.
- Produces structured audit log entries from metadata + state.

## Step-by-Step Instructions

1. **Define a metadata naming convention**: Use a `meta:` prefix for
   request-scoped metadata stored in session state:
   - `meta:trace_id` — distributed trace identifier.
   - `meta:client_version` — calling client version.
   - `meta:request_id` — unique per-request ID.
   - `meta:env` — deployment environment (`dev`, `staging`, `prod`).

2. **Inject metadata at session creation**:
   ```python
   import uuid
   session = await session_service.create_session(
       app_name="my_app",
       user_id=user_id,
       state={
           "meta:trace_id": str(uuid.uuid4()),
           "meta:client_version": "2.1.0",
           "meta:env": "production",
           "app:feature_search_v2": True,
       },
   )
   ```

3. **Read metadata in tool functions**:
   ```python
   trace_id = tool_context.state.get("meta:trace_id", "unknown")
   env = tool_context.state.get("meta:env", "prod")
   ```

4. **Use app-scoped metadata for feature flags**:
   ```python
   use_v2_search = tool_context.state.get("app:feature_search_v2", False)
   ```

5. **Log metadata with each tool call** for observability:
   ```python
   import logging
   logger = logging.getLogger(__name__)
   logger.info({
       "trace_id": tool_context.state.get("meta:trace_id"),
       "session_id": tool_context.session_id,
       "user_id": tool_context.user_id,
       "tool": "search_products",
       "event": "tool_call_start",
   })
   ```

6. **Use metadata for request routing** in callbacks or sub-agent selection.

7. **Never update `meta:` keys mid-session** — they are request-scoped constants.
   Only read them after they are set at session creation.

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_xyz",
  "state": {
    "meta:trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "meta:client_version": "2.1.0",
    "meta:env": "production",
    "meta:request_id": "req_7891234",
    "app:feature_search_v2": True,
    "app:api_endpoint": "https://api.example.com/v2"
  },
  "metadata": {}
}
```

## Output Format

```python
{
  "state_updated": False,
  "metadata_read": {
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "env": "production",
    "feature_search_v2": True
  },
  "audit_log_entry": {
    "trace_id": "4bf92f...",
    "session_id": "sess_abc123",
    "user_id": "user_xyz",
    "action": "search_products",
    "timestamp": 1700000000.0
  }
}
```

## Error Handling

- **`meta:trace_id` missing**: Generate a fallback UUID at the point of use.
  Log a warning — this indicates the session was created without metadata.
- **`app:` key missing** (feature flag not set): Default to `False` for boolean
  flags, `""` for string flags. Never raise on missing `app:` keys.
- **Invalid metadata value type**: Validate before use. If `meta:env` is not
  one of `["dev", "staging", "prod"]`, default to `"prod"` and log a warning.
- **Metadata too large** (inflating state): Limit `meta:` values to 256 chars.
  Do not store large payloads (JSON blobs) in `meta:` keys.
- **Metadata mutation mid-session**: Warn if `meta:` key is overwritten. Use
  assertion to detect:
  ```python
  assert "meta:trace_id" not in tool_context.state, "meta: keys are immutable"
  ```

## Examples

### Example 1 — Session initialization with full metadata

```python
import uuid
import time
from google.adk.sessions import DatabaseSessionService

session_service = DatabaseSessionService(db_url="sqlite+aiosqlite:///agent.db")

session = await session_service.create_session(
    app_name="my_app",
    user_id="user_001",
    state={
        # Request-scoped metadata
        "meta:trace_id": str(uuid.uuid4()),
        "meta:request_id": f"req_{int(time.time())}",
        "meta:client_version": "3.0.1",
        "meta:env": "production",
        "meta:platform": "web",
        # Application-scoped feature flags
        "app:feature_search_v2": True,
        "app:max_results": 10,
        "app:api_endpoint": "https://api.example.com/v2",
    },
)
```

### Example 2 — Reading metadata and feature flags in a tool

```python
from google.adk.tools import ToolContext
import logging

logger = logging.getLogger(__name__)

def search_products(query: str, tool_context: ToolContext) -> dict:
    """Searches products, using v2 API if feature flag is enabled.

    Args:
        query (str): Search query.
        tool_context (ToolContext): ADK tool context.

    Returns:
        dict: Search results.
    """
    trace_id = tool_context.state.get("meta:trace_id", "no-trace")
    env = tool_context.state.get("meta:env", "prod")
    use_v2 = tool_context.state.get("app:feature_search_v2", False)
    api_endpoint = tool_context.state.get("app:api_endpoint", "https://api.example.com/v1")

    logger.info({
        "trace_id": trace_id,
        "session_id": tool_context.session_id,
        "user_id": tool_context.user_id,
        "tool": "search_products",
        "query": query,
        "env": env,
        "use_v2": use_v2,
    })

    endpoint = f"{api_endpoint}/search{'_v2' if use_v2 else ''}"
    # ... perform search ...
    return {"results": [], "endpoint_used": endpoint, "trace_id": trace_id}
```

### Example 3 — Audit log entry from metadata

```python
import time
import json

def emit_audit_log(tool_name: str, action: str, tool_context: ToolContext) -> None:
    """Emits a structured audit log entry using session metadata."""
    entry = {
        "timestamp": time.time(),
        "trace_id": tool_context.state.get("meta:trace_id", "unknown"),
        "request_id": tool_context.state.get("meta:request_id", "unknown"),
        "session_id": tool_context.session_id,
        "user_id": tool_context.user_id,
        "tool": tool_name,
        "action": action,
        "env": tool_context.state.get("meta:env", "prod"),
    }
    import logging
    logging.getLogger("audit").info(json.dumps(entry))
```

## Edge Cases

- **Multi-turn session with changing metadata**: `meta:` keys should not change
  after session creation. If a new trace ID is needed per turn, use
  `meta:turn_trace_id` and update it each turn.
- **`app:` key conflict across agents**: Multiple agents sharing the same
  `app_name` share `app:` state. Use namespaced keys: `app:search_agent:max_results`.
- **Feature flag rollout mid-session**: The value is read from state at session
  start. Live flag changes won't apply until the next session creation.
- **Metadata in sub-agents**: Sub-agents invoked by a parent share the same
  `InvocationContext`, so all `meta:` and `app:` keys are visible to them.

## Integration Notes

- **`google-adk-conversation-state-tracking`**: `meta:` keys coexist with
  session state. Keep naming conventions distinct to avoid confusion.
- **Google Cloud Trace**: Use `meta:trace_id` to correlate ADK invocations with
  Cloud Trace spans for end-to-end distributed tracing.
- **`google-adk-session-management`**: Metadata is injected at session creation —
  apply this skill's initialization step within the session creation flow.
- **`before_agent_callback`**: Use to validate and normalize `meta:` keys at
  the start of each invocation before tool calls begin.
