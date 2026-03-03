---
name: google-adk-context-pruning
description: >
  Use this skill when a Google ADK agent's session.events history or
  accumulated state grows too large for the LLM's context window. Covers
  strategies for selectively removing old events, truncating state values,
  filtering events by type, and maintaining a minimal context footprint
  without losing critical conversation history.
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

This skill prevents context overflow in long-running ADK agent conversations
by selectively pruning `session.events` and `session.state` to stay within
LLM token limits. It establishes pruning strategies (sliding window, type-based
filtering, state key expiry) executed within ADK callbacks or custom runners,
ensuring the agent retains essential context while discarding stale data.

## When to Use

- When `session.events` grows beyond approximately 20–30 events and the LLM
  context window is at risk of overflow.
- When `session.state` contains accumulated data (lists, logs) that grows
  unboundedly over many turns.
- When a long-running multi-turn workflow must manage memory efficiently.
- Before injecting conversation history into an LLM call in a custom runner.

## When NOT to Use

- Do not prune state keys that are actively used by the next tool call.
- Do not prune events if the conversation replay (`google-adk-conversation-replay`)
  skill requires the full event history.
- Do not prune `user:` or `app:` scoped state — these are managed across sessions
  and should be preserved or explicitly expired.

## Google ADK Context

- **`session.events`**: Ordered list of `Event` objects. Each event may be a
  user message, agent response, tool call, or tool result. Pruning reduces the
  number of events sent to the LLM.
- **`session.state`**: Key-value scratchpad. Pruning removes keys that are no
  longer needed (e.g., `temp:` keys, intermediate results).
- **`tool_context.state`**: Read/write access in tool functions. Can be used to
  expire or reset accumulated state keys.
- **ADK Callbacks**: `before_agent_callback` is the recommended hook for
  pruning events before each model call. It receives `CallbackContext` with
  access to `session.events` and `session.state`.
- **ADK Execution Lifecycle**: Pruning should happen before the model call,
  not after. Use `before_agent_callback` for pre-call pruning.
- **`InvocationContext`**: Available in callbacks; provides `.session` access
  for reading and modifying events in-place.
- **Context compression**: ADK v0.3+ supports built-in context compression.
  See `google.github.io/adk-docs/context/compaction/`.

## Capabilities

- Implements sliding-window pruning: keep only the last N events.
- Implements type-based pruning: remove intermediate tool-call events, keep
  only user/agent messages.
- Removes expired `temp:` state keys after each invocation automatically.
- Truncates large string values in `session.state` to a character limit.
- Removes state keys matching a stale pattern (e.g., `temp:*`, `cache:*`).
- Applies pruning in `before_agent_callback` without modifying the persistent store.

## Step-by-Step Instructions

1. **Measure current context size** before each agent call. Count events and
   estimate token usage (rough estimate: 4 chars ≈ 1 token).

2. **Select a pruning strategy** based on the use case:
   - **Sliding window**: Keep the last N events.
   - **Type filter**: Remove all intermediate `tool_call`/`tool_response` events, keep user and agent messages.
   - **State key expiry**: Remove all `temp:` keys and any keys with age >
     a configured TTL.

3. **Implement the pruning in `before_agent_callback`**:
   ```python
   def prune_context(callback_context: CallbackContext) -> Optional[Content]:
       session = callback_context._invocation_context.session
       session.events = session.events[-20:]  # Keep last 20 events
       return None  # Allow agent to proceed
   ```

4. **Prune state keys** within the callback:
   ```python
   keys_to_remove = [k for k in callback_context.state if k.startswith("temp:")]
   for key in keys_to_remove:
       del callback_context.state[key]
   ```

5. **Truncate large state values**:
   ```python
   MAX_VALUE_LEN = 500
   for key, value in list(callback_context.state.items()):
       if isinstance(value, str) and len(value) > MAX_VALUE_LEN:
           callback_context.state[key] = value[:MAX_VALUE_LEN] + "...[truncated]"
   ```

6. **Attach the callback to the agent**:
   ```python
   agent = LlmAgent(
       name="my_agent",
       model="gemini-2.0-flash",
       before_agent_callback=prune_context,
   )
   ```

7. **Log pruning actions** for observability:
   ```python
   import logging
   logger = logging.getLogger(__name__)
   logger.info(f"Pruned events: {original_count} → {len(session.events)}")
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_xyz",
  "state": {
    "booking_step": "payment",
    "temp:raw_api_data": "{...large payload...}",
    "cache:search_results": "[...long list...]"
  },
  "metadata": {
    "max_events": 20,
    "max_state_value_length": 500,
    "prune_temp_keys": True
  }
}
```

## Output Format

```python
{
  "state_updated": True,
  "pruned_event_count": 8,
  "pruned_state_keys": ["temp:raw_api_data", "cache:search_results"],
  "truncated_state_keys": [],
  "remaining_event_count": 20
}
```

## Error Handling

- **Pruning removes a key needed by the next tool**: Identify critical keys
  before pruning and add them to an exclusion list.
- **`session.events` is read-only in the backend**: Do not attempt to modify
  events on the `SessionService`-retrieved session. Only prune in-memory in
  the callback context — the pruned list is used for the current model call only.
- **Pruning callback returns non-None**: Returning a `Content` object from
  `before_agent_callback` skips the agent's model call. Return `None` to allow
  normal execution after pruning.
- **Over-pruning (losing critical context)**: Always keep a minimum of the last
  5 events. Never prune the most recent user message.

## Examples

### Example 1 — Sliding-window event pruning callback

```python
from typing import Optional
from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content

MAX_EVENTS = 20

def sliding_window_pruner(callback_context: CallbackContext) -> Optional[Content]:
    """Keeps only the last MAX_EVENTS events before each agent call."""
    session = callback_context._invocation_context.session
    original_count = len(session.events)
    if original_count > MAX_EVENTS:
        session.events = session.events[-MAX_EVENTS:]
        import logging
        logging.getLogger(__name__).info(
            f"Context pruned: {original_count} → {len(session.events)} events"
        )
    return None  # Proceed with agent execution

agent = LlmAgent(
    name="assistant",
    model="gemini-2.0-flash",
    before_agent_callback=sliding_window_pruner,
)
```

### Example 2 — State key expiry in a tool

```python
from google.adk.tools import ToolContext

def cleanup_temp_state(tool_context: ToolContext) -> dict:
    """Removes all temp: prefixed keys from session state.

    Args:
        tool_context (ToolContext): ADK tool context.

    Returns:
        dict: Contains 'removed_keys' (list of str).
    """
    temp_keys = [k for k in tool_context.state if k.startswith("temp:")]
    for key in temp_keys:
        del tool_context.state[key]
    return {"removed_keys": temp_keys}
```

### Example 3 — Type-based event filtering

```python
from google.adk.events import Event

def type_filter_pruner(callback_context: CallbackContext) -> Optional[Content]:
    """Keeps only user messages and agent final responses; removes tool events."""
    session = callback_context._invocation_context.session
    session.events = [
        e for e in session.events
        if e.author in ("user", "agent") and not e.actions.tool_calls
    ]
    return None
```

## Edge Cases

- **Empty events list**: Do not prune — return immediately. Avoid index errors
  on empty slices.
- **All events are tool events**: If type-based filtering removes everything,
  keep the last 2 events unconditionally.
- **State value is a list that grows unboundedly**: Cap lists at a maximum
  length:
  ```python
  if isinstance(v, list) and len(v) > 50:
      callback_context.state[key] = v[-50:]
  ```
- **Concurrent pruning in parallel agents**: Each parallel agent gets its own
  callback invocation. Pruning is per-invocation — parallel agents share the
  session but each prunes its own view.

## Integration Notes

- **`before_agent_callback`**: The correct ADK hook for pre-call pruning.
  Return `None` to allow agent to proceed; return `Content` to short-circuit.
- **ADK Context Compression**: ADK has a built-in `context_compression` setting
  (v0.3+). Use this skill for custom pruning logic beyond what ADK provides.
- **`google-adk-context-summarization`**: Applied after pruning — summarizes
  the removed events and stores the summary in state for context continuity.
- **`google-adk-checkpointing`**: Should be called before pruning to save a
  checkpoint of the full context.
