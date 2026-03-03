---
name: google-adk-conversation-state-tracking
description: >
  Use this skill when a Google ADK agent needs to track, read, and update
  conversation-specific state across multiple turns within a single session.
  Covers reading from tool_context.state, writing session-scoped and
  temp-scoped state, and using output_key and EventActions.state_delta to
  persist state safely through the ADK event system.
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

This skill manages conversation-level state within a Google ADK session. It
enables agents to remember user inputs, track task progress across turns, and
make decisions based on accumulated context — all within the scope of a single
`Session` object. It governs the correct usage of `tool_context.state`,
`session.state`, and the ADK event system to ensure state is reliably tracked,
persisted, and never corrupted.

## When to Use

- When an agent must remember data from earlier turns (e.g., user preferences,
  workflow step, accumulated answers).
- When a tool function needs to read or update session-scoped state via
  `tool_context.state`.
- When implementing multi-turn task flows (e.g., booking wizard, form filling).
- When state must survive multiple tool calls within the same invocation.

## When NOT to Use

- Do not use for data that must survive across multiple sessions — use
  `google-adk-cross-session-memory` instead.
- Do not use `temp:` prefixed keys for data needed after the current invocation.
- Do not directly modify `session.state` on a `Session` object retrieved from
  `SessionService` outside a callback/tool context — this bypasses the event
  system.

## Google ADK Context

- **`tool_context.state`**: The primary interface for reading and writing state
  within tool functions. Changes are automatically captured into
  `EventActions.state_delta` and persisted via `session_service.append_event()`.
- **`tool_context.session_id`**: Identifies the current session. Read-only in tools.
- **`tool_context.user_id`**: Identifies the current user. Used with `user:` prefix
  keys for cross-session user state.
- **State Prefixes**:
  - No prefix → session-scoped, persists with `DatabaseSessionService`/`VertexAiSessionService`.
  - `user:` → user-scoped, shared across all sessions for this user.
  - `app:` → application-scoped, shared across all users.
  - `temp:` → invocation-scoped, discarded after the current invocation.
- **ADK Memory Model**: State is stored in `session.state` as a flat key-value
  dict. The `SessionService` merges scoped state from user/app/session stores.
- **ADK Execution Lifecycle**: State updates must occur within the ADK event
  lifecycle via `CallbackContext.state`, `ToolContext.state`, `output_key`, or
  `EventActions.state_delta`.

## Capabilities

- Read conversation state from `tool_context.state` within tool functions.
- Write session-scoped state updates that persist through the ADK event system.
- Use `output_key` on `LlmAgent` to auto-save final responses to state.
- Use `EventActions.state_delta` for multi-key or complex state updates.
- Use `{key}` placeholders in `LlmAgent.instruction` to inject state values.
- Track multi-turn workflow progress using session-scoped state keys.
- Use `temp:` prefix for intra-invocation scratchpad values.

## Step-by-Step Instructions

1. **Detect when this skill applies**: The agent needs to recall or update data
   from previous turns within the current session. Check if `tool_context.state`
   contains the relevant key before calling the tool.

2. **Read state from tool_context**:
   ```python
   value = tool_context.state.get("booking_step", "not_started")
   user_lang = tool_context.state.get("user:preferred_language", "en")
   ```
   Always provide a default to handle missing keys.

3. **Write/update state in tool_context**:
   ```python
   tool_context.state["booking_step"] = "payment_pending"
   tool_context.state["temp:raw_response"] = api_response
   ```
   The ADK framework captures these writes into `EventActions.state_delta`
   automatically on event append.

4. **Persist agent response to state** using `output_key` on `LlmAgent`:
   ```python
   agent = LlmAgent(name="summarizer", model="gemini-2.0-flash",
                    output_key="last_summary")
   ```

5. **Complex multi-key state updates** via `EventActions`:
   ```python
   from google.adk.events import Event, EventActions
   actions = EventActions(state_delta={
       "task_status": "complete",
       "user:session_count": current_count + 1,
   })
   await session_service.append_event(session, Event(..., actions=actions))
   ```

6. **Handle missing state**: Always use `.get(key, default)`. Never assume a
   key exists — it may be a new session.

7. **Inject state into agent instructions**:
   ```python
   agent = LlmAgent(instruction="Assist the user with step: {booking_step}")
   ```
   Use `{key?}` for optional state keys that may not exist.

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_xyz",
  "state": {
    "booking_step": "select_seat",
    "user:preferred_language": "en",
    "temp:raw_api_data": {...}
  },
  "metadata": {}
}
```

## Output Format

```python
{
  "state_updated": True,
  "state_key": "booking_step",
  "state_value": "payment_pending",
  "scope": "session"
}
```

## Error Handling

- **Missing `session_id`**: The `Runner` always provides `session_id` via
  `tool_context`. If absent, raise `RuntimeError("No session_id in tool_context")`.
- **Missing state key**: Use `.get(key, default)` — never `state[key]` directly.
- **Corrupted state (wrong type)**: Validate with `isinstance()` before using.
  Return an error dict if the type is unexpected.
- **Serialization errors**: Only store JSON-serializable types. If a complex
  object must be tracked, store its ID/reference, not the object itself.
- **Concurrent state updates**: Use `append_event()` — it is thread-safe. Never
  mutate `session.state` directly on a retrieved `Session` object.
- **Memory storage failure**: Wrap `append_event()` in try/except. On failure,
  log the error and re-raise to the runner for retry.

## Examples

### Example 1 — Reading and writing state in a tool

```python
from google.adk.tools import ToolContext

def advance_booking_step(tool_context: ToolContext) -> dict:
    """Advances the booking workflow to the next step.

    Args:
        tool_context (ToolContext): ADK tool context with state access.

    Returns:
        dict: Contains 'current_step' (str) and 'next_step' (str).
    """
    STEPS = ["select_flight", "select_seat", "enter_details", "payment", "confirmed"]

    current = tool_context.state.get("booking_step", STEPS[0])
    idx = STEPS.index(current) if current in STEPS else 0
    next_step = STEPS[min(idx + 1, len(STEPS) - 1)]

    tool_context.state["booking_step"] = next_step
    tool_context.state["temp:step_transition_ts"] = __import__("time").time()

    return {"current_step": current, "next_step": next_step}
```

### Example 2 — Agent with output_key and state injection

```python
from google.adk.agents import LlmAgent

booking_agent = LlmAgent(
    name="booking_agent",
    model="gemini-2.0-flash",
    instruction=(
        "You are helping a user book a flight. "
        "Current step: {booking_step}. "
        "User preferred language: {user:preferred_language?}."
    ),
    tools=[advance_booking_step],
    output_key="last_agent_response",
)
```

### Example 3 — Manual state delta update

```python
from google.adk.events import Event, EventActions
import time

state_changes = {
    "booking_step": "confirmed",
    "user:total_bookings": (tool_context.state.get("user:total_bookings", 0) + 1),
    "user:last_booking_ts": time.time(),
}
actions = EventActions(state_delta=state_changes)
event = Event(invocation_id="inv_booking_complete", author="agent", actions=actions)
await session_service.append_event(session, event)
```

## Edge Cases

- **New session (empty state)**: All `.get()` calls return defaults. Initialize
  required keys in the `create_session()` call: `state={"booking_step": "select_flight"}`.
- **Expired session**: `get_session()` returns `None`. Always check before use.
  Recreate the session if expired.
- **State conflict (concurrent writes)**: `append_event()` is atomic per event.
  Never call it concurrently for the same session.
- **`temp:` key read after invocation**: `temp:` keys are discarded after the
  invocation. Do not read them in a subsequent turn — they will not exist.
- **Large state values**: Keep state values small and serializable. Never store
  raw API responses — extract and store only needed fields.

## Integration Notes

- **`tool_context.state`**: Primary interface in tool functions. Automatically
  feeds into `EventActions.state_delta` on event append.
- **`InMemorySessionService`**: No persistence. All state is lost on restart.
  Use for development only.
- **`DatabaseSessionService`**: Persists session, user, and app state. Requires
  async DB driver (e.g., `aiosqlite`, `asyncpg`).
- **`VertexAiSessionService`**: Google Cloud-managed persistence. Production
  standard for GCP deployments.
- **`google-adk-session-management` skill**: Governs session lifecycle (create,
  resume, delete). Apply before this skill.
- **`google-adk-context-injection` skill**: Governs injection of state into
  agent instructions. Works in conjunction with this skill.
