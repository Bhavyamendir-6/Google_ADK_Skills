---
name: google-adk-conversation-replay
description: >
  Use this skill when a Google ADK agent needs to reconstruct, replay, or
  audit a previous conversation using session.events history. Covers reading
  and iterating over Event objects, reconstructing conversation turns,
  replaying tool calls deterministically, and generating audit transcripts
  from persisted session data.
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

This skill enables reconstruction and replay of past ADK conversations from
`session.events`. It covers reading the event history from a persisted
`Session`, reconstructing the conversation transcript, replaying tool calls
for debugging, and generating human-readable audit logs. It is used for
debugging agent behavior, compliance auditing, and session recovery after
failures.

## When to Use

- When debugging why an agent produced a specific response in a past session.
- When generating a human-readable transcript of a completed session.
- When auditing tool calls made during a session for compliance.
- When recovering a failed session by replaying events up to the failure point.
- When re-running an agent against a recorded conversation for regression testing.

## When NOT to Use

- Do not use for real-time state access — use `tool_context.state` instead.
- Do not replay events in production sessions — only replay in debug or test contexts.
- Do not use `InMemorySessionService` for replay — events are not persisted.
- Do not replay events to the live LLM without careful filtering — this may
  produce duplicate side effects (e.g., sending emails again).

## Google ADK Context

- **`session.events`**: Ordered list of `Event` objects. Each event contains:
  - `event.author`: `"user"`, `"agent"`, `"tool"`, or `"system"`.
  - `event.content`: `Content` object with `parts` (text, function_call, function_response).
  - `event.invocation_id`: Groups events belonging to the same agent invocation.
  - `event.timestamp`: Unix timestamp of the event.
  - `event.actions.state_delta`: State changes applied by this event.
  - `event.is_final_response()`: True if this is an agent's final text response.
- **`session.state`**: The final accumulated state after all events have been applied.
- **`SessionService.get_session()`**: Retrieves the session with all events and
  current state. Requires a persistent `SessionService` backend.
- **ADK Execution Lifecycle**: Events are immutable after `append_event()`.
  Replay reads but never modifies the event list.

## Capabilities

- Reads `session.events` from `DatabaseSessionService` or `VertexAiSessionService`.
- Generates a human-readable conversation transcript from event history.
- Extracts all tool calls and their results from the event history.
- Reconstructs state at any point in the conversation by replaying `state_delta`.
- Filters events by author, type, or invocation ID for targeted analysis.
- Exports event history as structured JSON for external audit systems.

## Step-by-Step Instructions

1. **Retrieve the target session** from a persistent `SessionService`:
   ```python
   session = await session_service.get_session(
       app_name="my_app", user_id="user_123", session_id="sess_abc"
   )
   assert session is not None, "Session not found"
   ```

2. **Iterate `session.events`** in chronological order:
   ```python
   for event in session.events:
       process_event(event)
   ```

3. **Extract conversation turns** (user and agent messages only):
   ```python
   turns = [
       {"author": e.author, "text": e.content.parts[0].text, "ts": e.timestamp}
       for e in session.events
       if e.content and e.content.parts and hasattr(e.content.parts[0], "text")
       and e.author in ("user", "agent") and e.is_final_response() or e.author == "user"
   ]
   ```

4. **Extract tool calls and results**:
   ```python
   tool_calls = []
   for e in session.events:
       if e.content and e.content.parts:
           for part in e.content.parts:
               if hasattr(part, "function_call") and part.function_call:
                   tool_calls.append({"name": part.function_call.name, "args": dict(part.function_call.args)})
               if hasattr(part, "function_response") and part.function_response:
                   tool_calls.append({"name": part.function_response.name, "result": part.function_response.response})
   ```

5. **Reconstruct state at event N** by replaying `state_delta`:
   ```python
   reconstructed_state = {}
   for event in session.events[:n]:
       if event.actions and event.actions.state_delta:
           reconstructed_state.update(event.actions.state_delta)
   ```

6. **Generate a structured audit log**:
   ```python
   audit = {"session_id": session.id, "user_id": session.user_id, "turns": turns, "tool_calls": tool_calls}
   ```

7. **Export the audit log** to JSON or a logging system.

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_xyz",
  "state": {},              # Retrieved from SessionService
  "metadata": {
    "export_format": "json" | "text",
    "replay_up_to_event": None,   # None = full replay
    "filter_author": None         # None = all authors
  }
}
```

## Output Format

Conversation transcript:
```python
{
  "session_id": "sess_abc123",
  "user_id": "user_xyz",
  "event_count": 12,
  "turns": [
    {"author": "user", "text": "Book a flight to London", "timestamp": 1700000000.0},
    {"author": "agent", "text": "I found 3 flights. Which date works?", "timestamp": 1700000005.0}
  ],
  "tool_calls": [
    {"name": "search_flights", "args": {"destination": "London"}, "result": {"flights": [...]}},
  ],
  "final_state": {"booking_step": "confirmed"}
}
```

## Error Handling

- **Session not found** (`get_session()` returns `None`): Raise `ValueError` with
  session ID and user ID. Do not attempt replay.
- **`session.events` is empty**: Return an empty transcript — do not raise.
- **Event with no `content`**: Skip the event. Not all events (e.g., internal
  system events) carry content.
- **`state_delta` contains non-serializable types**: Filter out non-JSON-safe
  values during state reconstruction. Log a warning.
- **`InMemorySessionService` used**: Events are lost on restart. Raise a clear
  error advising migration to a persistent backend.

## Examples

### Example 1 — Full conversation transcript Generator

```python
import json
from google.adk.sessions import DatabaseSessionService

async def generate_transcript(session_service, app_name: str, user_id: str, session_id: str) -> dict:
    """Generates a structured conversation transcript from session events."""
    session = await session_service.get_session(
        app_name=app_name, user_id=user_id, session_id=session_id
    )
    if session is None:
        raise ValueError(f"Session {session_id} not found for user {user_id}")

    turns, tool_calls = [], []

    for event in session.events:
        if not event.content or not event.content.parts:
            continue
        for part in event.content.parts:
            if hasattr(part, "text") and part.text:
                if event.author in ("user", "agent"):
                    turns.append({
                        "author": event.author,
                        "text": part.text,
                        "timestamp": event.timestamp,
                        "invocation_id": event.invocation_id,
                    })
            if hasattr(part, "function_call") and part.function_call:
                tool_calls.append({
                    "type": "call",
                    "name": part.function_call.name,
                    "args": dict(part.function_call.args),
                    "invocation_id": event.invocation_id,
                })
            if hasattr(part, "function_response") and part.function_response:
                tool_calls.append({
                    "type": "result",
                    "name": part.function_response.name,
                    "result": part.function_response.response,
                    "invocation_id": event.invocation_id,
                })

    return {
        "session_id": session.id,
        "user_id": session.user_id,
        "event_count": len(session.events),
        "turns": turns,
        "tool_calls": tool_calls,
        "final_state": dict(session.state),
    }
```

### Example 2 — State reconstruction at event N

```python
def reconstruct_state_at(session, event_index: int) -> dict:
    """Reconstructs session.state as it was after event N."""
    state = {}
    for event in session.events[:event_index + 1]:
        if event.actions and event.actions.state_delta:
            # Filter out temp: keys — they don't persist
            delta = {k: v for k, v in event.actions.state_delta.items()
                     if not k.startswith("temp:")}
            state.update(delta)
    return state
```

### Example 3 — Export audit log to JSON file

```python
import json
import os

async def export_session_audit(session_service, app_name, user_id, session_id, output_dir="/tmp"):
    transcript = await generate_transcript(session_service, app_name, user_id, session_id)
    output_path = os.path.join(output_dir, f"audit_{session_id}.json")
    with open(output_path, "w") as f:
        json.dump(transcript, f, indent=2, default=str)
    return output_path
```

## Edge Cases

- **Session with millions of events** (very long session): Use pagination if
  the `SessionService` supports it. Otherwise, process events in batches of 100.
- **Events from parallel agents** (same `invocation_id`, different authors):
  Group by `invocation_id` then by `timestamp` for correct ordering.
- **Corrupted event** (missing `invocation_id`): Skip with a warning. Do not
  abort the full replay.
- **`temp:` keys in `state_delta`**: Exclude from state reconstruction — they
  were never persisted.
- **Tool calls in replay trigger real side effects**: Never call tool functions
  during replay. Only read tool call/response data from events — do not re-execute.

## Integration Notes

- **`DatabaseSessionService` / `VertexAiSessionService`**: Required for replay.
  Events are only accessible from persistent backends.
- **`google-adk-checkpointing`**: Checkpoints create synthetic events in the
  history. Filter checkpoint events by `event.author == "checkpoint"` during
  replay to distinguish them from conversation events.
- **`google-adk-context-summarization`**: The summary event is stored in
  `state_delta`. During replay, it appears as a state update event.
- **ADK Event system**: Events are append-only and immutable after
  `append_event()`. Replay is always read-only.
