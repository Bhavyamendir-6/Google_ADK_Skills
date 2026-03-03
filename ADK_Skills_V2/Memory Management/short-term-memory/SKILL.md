---
name: short-term-memory
description: >
  Use this skill when a Google ADK agent needs to store and retrieve transient
  data within a single session — data that must persist across multiple turns
  of a conversation but does not need to survive beyond the session lifetime.
  Covers using tool_context.state and session-scoped state keys for in-session
  memory, reading and writing ephemeral context, and managing state key
  namespacing to avoid collisions.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: memory-management
---

## Purpose

This skill implements short-term (session-scoped) memory for Google ADK agents
using `tool_context.state`. Short-term memory holds data that must be available
across multiple turns within a single conversation session — such as the user's
current workflow step, intermediate tool results, collected form fields, and
conversation-specific preferences. All session-scoped state in ADK is automatically
persisted by the session service backend and is transparently available to the
agent and all its tools throughout the session.

## When to Use

- When a tool needs to store intermediate results for use by another tool in
  a subsequent turn of the same session.
- When the agent tracks a multi-step workflow (e.g., booking step: search →
  confirm → payment) and needs to know the current step on each turn.
- When user input is collected across multiple turns (e.g., a form) and must
  be accumulated before processing.
- When context computed in one turn (e.g., a search result) must be available
  in the next turn without recomputing.
- When caching expensive tool results within a session to avoid redundant calls.

## When NOT to Use

- Do not use session-scoped state for data that must survive across sessions —
  use `user:` prefixed keys or a persistent backend.
- Do not store large binary blobs (images, files) in `tool_context.state` — it
  is designed for lightweight structured data (strings, dicts, numbers).
- Do not use session state as a substitute for tool return values — return data
  from tools directly and write to state only when cross-turn persistence is needed.

## Google ADK Context

- **`tool_context.state`**: A dict-like object available in every tool function.
  Reads and writes are persisted by the session service backend across turns.
- **Session-scoped keys**: Unnamespaced keys (e.g., `state["booking_step"]`) are
  session-scoped — visible only within the current session.
- **`temp:` prefix**: `state["temp:key"]` is cleared at the end of each invocation.
  Use for truly ephemeral within-turn data.
- **`user:` prefix**: Cross-session user-scoped state. Persists across sessions
  for the same user_id.
- **`app:` prefix**: Application-global state. Shared across all users and sessions.
- **`tool_context.session_id`**: Unique identifier for the current session. Use
  as a correlation key for logging.
- **`tool_context.user_id`**: User identifier. Use for `user:`-scoped state.
- **ADK Agent Execution Lifecycle**: State written by a tool in turn N is
  available in turn N+1 without any additional code.
- **Session service backends**: `InMemorySessionService` (dev/test),
  `DatabaseSessionService` (production), `VertexAiSessionService` (GCP-native).

## Capabilities

- Stores ephemeral per-session data in `tool_context.state`.
- Reads session state from any tool within the same session.
- Tracks multi-step workflow progress across turns.
- Accumulates user input across multiple conversation turns.
- Caches tool results within a session.
- Uses namespaced keys to prevent state collisions.
- Uses `temp:` keys for within-turn ephemeral data.

## Step-by-Step Instructions

1. **Identify the data to persist** across turns. Confirm it is session-scoped
   (not cross-session) and lightweight (< 10KB).

2. **Write to session state** in a tool function using a namespaced key:
   ```python
   def set_booking_step(step: str, tool_context: ToolContext) -> dict:
       tool_context.state["booking_step"] = step
       tool_context.state["booking_step_timestamp"] = time.time()
       return {"stored": True, "step": step}
   ```

3. **Read from session state** in the same or a subsequent tool call:
   ```python
   def get_booking_step(tool_context: ToolContext) -> dict:
       step = tool_context.state.get("booking_step", "search")
       return {"current_step": step}
   ```

4. **Inject state into the agent instruction** via `{key?}` placeholders:
   ```python
   instruction = "Current booking step: {booking_step?}. Act accordingly."
   ```

5. **Clear state when it is no longer needed** to avoid stale data:
   ```python
   del tool_context.state["booking_step"]
   ```

6. **Use `temp:` prefix** for within-turn ephemeral data (auto-cleared after each turn):
   ```python
   tool_context.state["temp:search_query_hash"] = hash_query(query)
   ```

7. **Validate state type** before reading to handle corrupted or missing values:
   ```python
   step = tool_context.state.get("booking_step")
   if not isinstance(step, str):
       step = "search"  # default
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "short_term",
  "content": "booking_step=confirm",
  "metadata": {"timestamp": 1709900000, "turn": 3},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "sess_abc123:booking_step",
  "memory_retrieved": [{"key": "booking_step", "value": "confirm"}],
  "memory_injected": True
}
```

## Error Handling

- **Missing key** (`state.get()` returns None): Always use `.get(key, default)` — 
  never direct `state[key]` access for reading. Provide sensible defaults.
- **State value type mismatch** (expected str, got dict): Validate type on read:
  `val = str(state.get("key", ""))`.
- **State write failure** (session backend error): ADK raises an exception.
  Catch, log, and return a graceful error dict from the tool.
- **`temp:` key persisting** (session service not clearing it): In custom session
  backends, implement `temp:` key clearing at invocation end.
- **Concurrent write collision** (parallel tools writing the same key):
  In `ParallelAgent`, use distinct key names per sub-agent. Last write wins.

## Examples

### Example 1 — Multi-step workflow state tracking

```python
import time
from google.adk.tools import ToolContext

VALID_STEPS = {"search", "confirm", "payment", "complete"}

def advance_booking_step(new_step: str, tool_context: ToolContext) -> dict:
    """Advances the booking workflow to the specified step.

    Args:
        new_step (str): Target step — one of: search, confirm, payment, complete.
        tool_context (ToolContext): ADK tool context with session state.
    Returns:
        dict: Contains 'previous_step' (str), 'current_step' (str).
    """
    if new_step not in VALID_STEPS:
        return {"error": f"Invalid step: {new_step}", "valid_steps": list(VALID_STEPS)}
    previous = tool_context.state.get("booking_step", "search")
    tool_context.state["booking_step"] = new_step
    tool_context.state["booking_step_updated_at"] = time.time()
    return {"previous_step": previous, "current_step": new_step}


def get_booking_step(tool_context: ToolContext) -> dict:
    """Gets the current booking workflow step from session state.

    Args:
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'current_step' (str).
    """
    step = tool_context.state.get("booking_step", "search")
    if step not in VALID_STEPS:
        step = "search"
    return {"current_step": step}
```

### Example 2 — Accumulating form fields across turns

```python
import json
from google.adk.tools import ToolContext

def save_passenger_field(field_name: str, field_value: str, tool_context: ToolContext) -> dict:
    """Saves a passenger form field to session state for multi-turn collection.

    Args:
        field_name (str): Form field name (e.g., 'first_name', 'passport_number').
        field_value (str): Field value provided by the user.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'saved_fields' (list), 'pending_fields' (list).
    """
    REQUIRED_FIELDS = {"first_name", "last_name", "passport_number", "date_of_birth"}
    passenger_data = tool_context.state.get("passenger_data", {})
    passenger_data[field_name] = field_value
    tool_context.state["passenger_data"] = passenger_data

    saved = set(passenger_data.keys())
    pending = list(REQUIRED_FIELDS - saved)
    return {"saved_fields": list(saved), "pending_fields": pending, "complete": len(pending) == 0}
```

### Example 3 — Session-scoped result caching

```python
import hashlib
import json
import time
from google.adk.tools import ToolContext

CACHE_TTL_SECONDS = 300

def search_with_session_cache(query: str, tool_context: ToolContext) -> dict:
    """Searches with session-scoped caching to avoid redundant API calls.

    Args:
        query (str): Search query.
        tool_context (ToolContext): ADK tool context with state caching.
    Returns:
        dict: Contains 'results' (list), 'cached' (bool).
    """
    cache_key = f"cache:search:{hashlib.md5(query.lower().encode()).hexdigest()}"
    cached_entry = tool_context.state.get(cache_key)
    if cached_entry and isinstance(cached_entry, dict):
        if time.time() - cached_entry.get("ts", 0) < CACHE_TTL_SECONDS:
            return {"results": cached_entry["results"], "cached": True}

    results = _do_real_search(query)
    tool_context.state[cache_key] = {"results": results, "ts": time.time()}
    return {"results": results, "cached": False}

def _do_real_search(query):
    return []  # Placeholder for real search implementation
```

## Edge Cases

- **Empty session** (first turn, no state): All `.get()` calls return None or
  the provided default. Design tools to handle empty state gracefully.
- **Large state value** (> 64KB): Session service backends may impose size limits.
  Store large data externally (GCS, database) and store only the reference in state.
- **State key naming collision** (two tools use the same key): Use tool-namespaced
  keys: `f"tool:{tool_name}:{key}"` to avoid accidental overwrites.
- **Session expires mid-workflow**: `DatabaseSessionService` sessions expire
  based on TTL. Implement session resumption by storing a job/reference ID that
  can be used to restore context.
- **State modification from before_agent_callback**: State written in
  `before_agent_callback` is available to all tools in that turn.

## Integration Notes

- **`google-adk-long-term-memory`**: For data that must persist across sessions,
  use `user:` prefixed keys or a persistent memory backend.
- **`google-adk-context-window-management`**: If accumulated session state grows
  too large to inject into the instruction, apply compression/summarization.
- **`google-adk-memory-persistence`**: Covers the session service backends that
  persist `tool_context.state` across server restarts.
- **`{key?}` injection**: Session state values are directly injectable into
  `LlmAgent.instruction` strings — a zero-code mechanism for context injection.
