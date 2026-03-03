---
name: google-adk-checkpointing
description: >
  Use this skill when a Google ADK agent needs to save the current session
  state as a named checkpoint that can be restored if a subsequent workflow
  step fails. Covers creating checkpoint events via EventActions.state_delta,
  restoring from checkpoints using session.events replay, and integrating
  checkpointing into multi-step agent workflows.
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

This skill implements session state checkpointing in Google ADK agents —
saving named snapshots of `session.state` at critical workflow milestones and
enabling restoration to a prior checkpoint if a subsequent step fails. It
uses ADK's `EventActions.state_delta` to persist checkpoint data within the
event history, ensuring checkpoints are part of the auditable session record
without external storage dependencies.

## When to Use

- Before executing an irreversible or high-risk tool call (e.g., payment
  processing, data deletion).
- In multi-step workflows where a step failure should roll back to the last
  safe state.
- When implementing long-horizon agent tasks that may require resumption after
  failure.
- When you need a named rollback point before an experimental or uncertain
  workflow branch.

## When NOT to Use

- Do not use `InMemorySessionService` for checkpointing — checkpoints are lost
  on restart.
- Do not checkpoint after every single turn — only at meaningful milestones.
- Do not use checkpointing as a substitute for proper transaction management
  in external systems.
- Do not checkpoint `temp:` prefixed keys — they are invocation-scoped and
  are NOT persisted.

## Google ADK Context

- **`EventActions.state_delta`**: The ADK mechanism for atomically persisting
  state changes. Checkpoints are stored as state entries within a dedicated
  event.
- **`session.events`**: Append-only list. Checkpoints are stored as events with
  a recognizable `author` (e.g., `"checkpoint"`) for later retrieval.
- **`session_service.append_event()`**: The only safe way to write checkpoints.
  Direct `session.state` mutation is unsafe and won't persist.
- **`tool_context.state`**: Used within tool functions to write checkpoint
  markers into state.
- **`session.state["checkpoint:<name>"]`**: Naming convention for checkpoint
  entries. Stores a JSON-serialized snapshot of the relevant state subset.
- **ADK Execution Lifecycle**: Checkpoint creation occurs within a tool function
  or callback, using `tool_context.state` or `EventActions.state_delta`.
- **Restoration**: Replay `state_delta` values from checkpoint events to
  reconstruct the saved state (see `google-adk-conversation-replay`).

## Capabilities

- Creates named checkpoints of `session.state` at workflow milestones via tools.
- Stores checkpoints as `checkpoint:<name>` keys in state, persisted by the event system.
- Restores session state from a named checkpoint by reading the stored snapshot.
- Lists available checkpoints from `session.state` keys.
- Integrates checkpointing into before-step and after-step tool call hooks.
- Supports rollback by overwriting current state keys with checkpoint values.

## Step-by-Step Instructions

1. **Identify checkpoint moments** in your workflow:
   - Before an irreversible action.
   - After completing a major workflow phase.
   - Before a branching decision point.

2. **Create a `save_checkpoint` tool function**:
   ```python
   def save_checkpoint(name: str, tool_context: ToolContext) -> dict:
       snapshot = {k: v for k, v in tool_context.state.items()
                   if not k.startswith("temp:") and not k.startswith("checkpoint:")}
       import json, time
       tool_context.state[f"checkpoint:{name}"] = json.dumps({
           "snapshot": snapshot, "timestamp": time.time()
       })
       return {"checkpoint_saved": True, "name": name}
   ```

3. **Register the checkpoint tool** on the agent:
   ```python
   agent = LlmAgent(name="workflow_agent", model="gemini-2.0-flash",
                    tools=[save_checkpoint, restore_checkpoint, ...])
   ```

4. **Instruct the agent to checkpoint** at key milestones:
   ```python
   instruction = (
       "Before calling payment_processor, always call save_checkpoint('pre_payment'). "
       "If payment fails, call restore_checkpoint('pre_payment')."
   )
   ```

5. **Create a `restore_checkpoint` tool function** to roll back state:
   ```python
   def restore_checkpoint(name: str, tool_context: ToolContext) -> dict:
       import json
       checkpoint_data = tool_context.state.get(f"checkpoint:{name}")
       if checkpoint_data is None:
           return {"error": f"Checkpoint '{name}' not found.", "error_type": "not_found"}
       data = json.loads(checkpoint_data)
       for key, value in data["snapshot"].items():
           tool_context.state[key] = value
       return {"restored": True, "name": name, "snapshot_timestamp": data["timestamp"]}
   ```

6. **List available checkpoints**:
   ```python
   checkpoints = [k.replace("checkpoint:", "") for k in tool_context.state
                  if k.startswith("checkpoint:")]
   ```

7. **Verify the checkpoint persisted** by reading it back from state after the
   tool call completes.

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_xyz",
  "state": {
    "booking_step": "payment_pending",
    "selected_flight": "AA123",
    "user:name": "Alice",
    "checkpoint:pre_payment": "{\"snapshot\": {\"booking_step\": \"seat_selected\"}, \"timestamp\": 1700000000.0}"
  },
  "metadata": {
    "checkpoint_name": "pre_payment"
  }
}
```

## Output Format

On save:
```python
{
  "state_updated": True,
  "state_key": "checkpoint:pre_payment",
  "checkpoint_saved": True,
  "name": "pre_payment",
  "snapshot_keys": ["booking_step", "selected_flight", "passenger_name"]
}
```

On restore:
```python
{
  "state_updated": True,
  "restored": True,
  "name": "pre_payment",
  "snapshot_timestamp": 1700000000.0,
  "restored_keys": ["booking_step", "selected_flight"]
}
```

## Error Handling

- **Checkpoint not found on restore**: Return `{"error": "Checkpoint 'pre_payment' not found.", "error_type": "not_found", "recoverable": False}`.
- **Checkpoint JSON corrupted**: Catch `json.JSONDecodeError`. Return error and
  list available checkpoints for fallback.
- **Snapshot too large** (> 10KB): Checkpoint only the critical state keys
  needed for rollback. Pass an explicit `keys_to_checkpoint: list[str]` param.
- **`temp:` keys accidentally checkpointed**: Filter them out at save time.
  Never include `temp:` keys in checkpoint snapshots.
- **Restore overwrites newer state**: Restoration is destructive. Confirm
  explicitly (add `confirm: bool = False` param to `restore_checkpoint`).
- **`InMemorySessionService`**: Checkpoints will not survive restart. Raise a
  warning if the backend is detected as in-memory.

## Examples

### Example 1 — Complete checkpointing tool pair

```python
import json
import time
from google.adk.tools import ToolContext

def save_checkpoint(name: str, tool_context: ToolContext) -> dict:
    """Saves a named checkpoint of the current session state.

    Args:
        name (str): Checkpoint name, e.g., 'pre_payment'.
        tool_context (ToolContext): ADK tool context.

    Returns:
        dict: Contains 'checkpoint_saved' (bool) and 'name' (str).
    """
    snapshot = {
        k: v for k, v in tool_context.state.items()
        if not k.startswith("temp:") and not k.startswith("checkpoint:")
    }
    checkpoint_key = f"checkpoint:{name}"
    tool_context.state[checkpoint_key] = json.dumps({
        "snapshot": snapshot,
        "timestamp": time.time(),
        "session_id": tool_context.session_id,
    })
    return {"checkpoint_saved": True, "name": name, "snapshot_keys": list(snapshot.keys())}


def restore_checkpoint(name: str, confirm: bool = False, tool_context: ToolContext = None) -> dict:
    """Restores session state from a named checkpoint.

    Args:
        name (str): Name of the checkpoint to restore.
        confirm (bool): Must be True to execute the restore. Defaults to False (preview).
        tool_context (ToolContext): ADK tool context.

    Returns:
        dict: Restore result or preview.
    """
    checkpoint_key = f"checkpoint:{name}"
    raw = tool_context.state.get(checkpoint_key)
    if raw is None:
        available = [k.replace("checkpoint:", "") for k in tool_context.state if k.startswith("checkpoint:")]
        return {"error": f"Checkpoint '{name}' not found.", "available": available, "error_type": "not_found", "recoverable": False}

    try:
        data = json.loads(raw)
    except json.JSONDecodeError:
        return {"error": f"Checkpoint '{name}' is corrupted.", "error_type": "unexpected", "recoverable": False}

    snapshot = data["snapshot"]
    if not confirm:
        return {"requires_confirmation": True, "name": name, "will_restore_keys": list(snapshot.keys()), "snapshot_timestamp": data["timestamp"]}

    for key, value in snapshot.items():
        tool_context.state[key] = value

    return {"restored": True, "name": name, "snapshot_timestamp": data["timestamp"], "restored_keys": list(snapshot.keys())}
```

### Example 2 — Workflow agent with checkpointing instruction

```python
from google.adk.agents import LlmAgent

workflow_agent = LlmAgent(
    name="booking_workflow",
    model="gemini-2.0-flash",
    instruction=(
        "You manage a flight booking workflow.\n"
        "Step 1: Search flights → Step 2: Select seat → Step 3: Process payment.\n"
        "BEFORE calling process_payment, always call save_checkpoint('pre_payment').\n"
        "If process_payment fails, call restore_checkpoint('pre_payment', confirm=True) to rollback.\n"
        "Current step: {booking_step?}."
    ),
    tools=[save_checkpoint, restore_checkpoint, search_flights, select_seat, process_payment],
)
```

### Example 3 — Listing checkpoints in a tool

```python
def list_checkpoints(tool_context: ToolContext) -> dict:
    """Lists all available checkpoints in the current session.

    Args:
        tool_context (ToolContext): ADK tool context.

    Returns:
        dict: Contains 'checkpoints' (list of dicts with name and timestamp).
    """
    checkpoints = []
    for key, value in tool_context.state.items():
        if key.startswith("checkpoint:"):
            name = key.replace("checkpoint:", "")
            try:
                data = json.loads(value)
                checkpoints.append({"name": name, "timestamp": data.get("timestamp")})
            except Exception:
                checkpoints.append({"name": name, "timestamp": None, "corrupted": True})
    return {"checkpoints": checkpoints, "count": len(checkpoints)}
```

## Edge Cases

- **Restoring in the middle of an active workflow**: Restoration reverts state
  keys but not `session.events`. The event history is immutable. Design rollback
  to only revert state, not re-execute past events.
- **Multiple checkpoints in sequence**: Each saves the full state snapshot at
  that moment. Old checkpoints remain in state until explicitly deleted.
- **Checkpoint key collision**: If `checkpoint:pre_payment` already exists,
  overwrite it (latest timestamp wins). Add `overwrite: bool = True` parameter
  to signal intent.
- **Very large state to checkpoint**: Implement selective checkpointing —
  pass `keys: list[str]` to `save_checkpoint` to snapshot only critical keys.
- **Checkpoint name with special chars**: Validate name: `[a-zA-Z0-9_-]` only.
  Reject others.

## Integration Notes

- **`google-adk-conversation-state-tracking`**: Checkpoints are stored as
  regular state keys (`checkpoint:<name>`) and governed by the same state
  update rules.
- **`google-adk-session-management`**: Must use `DatabaseSessionService` or
  `VertexAiSessionService` for checkpoints to survive restarts.
- **`google-adk-conversation-replay`**: Checkpoints appear as `state_delta`
  entries with `checkpoint:<name>` keys in the replayed event history.
- **`google-adk-context-pruning`**: Exclude `checkpoint:` keys from pruning —
  they must remain available for restoration throughout the session.
