---
name: long-term-memory
description: >
  Use this skill when a Google ADK agent needs to store and retrieve data that
  persists across multiple sessions for the same user. Covers using user:-prefixed
  tool_context.state keys for cross-session storage, integrating with persistent
  session service backends, and building a long-term memory layer using vector
  stores or databases for retrieval-augmented agent reasoning.
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

This skill implements long-term memory for Google ADK agents — data that must
persist across multiple sessions for the same user. Long-term memory enables
agents to remember user preferences, past interactions, accumulated knowledge,
and completed tasks across separate conversation sessions. In ADK, the primary
mechanism is `user:` prefixed state keys (automatically shared across sessions
for the same `user_id`) backed by a persistent session service. For semantic
retrieval of past context, this skill integrates with vector stores.

## When to Use

- When the agent must remember user preferences, name, or settings across sessions
  (e.g., "I prefer aisle seats" stored permanently).
- When the agent accumulates knowledge from multiple sessions (e.g., user's
  complete purchase history).
- When the agent must recognize returning users and personalize based on past
  interactions.
- When completed task records must be archived for future reference.
- When building a user profile that grows richer with each session.

## When NOT to Use

- Do not use long-term memory for session-scoped data — use unnamespaced state keys.
- Do not store sensitive PII in `user:` state without appropriate access controls
  and encryption.
- Do not use `user:` state for binary data or large payloads — keep values < 64KB.

## Google ADK Context

- **`user:` prefix**: `tool_context.state["user:key"]` is persisted across
  all sessions for the same `tool_context.user_id`. This is ADK's built-in
  long-term cross-session memory mechanism.
- **`tool_context.user_id`**: The stable user identifier. All `user:` keys are
  scoped to this ID.
- **`DatabaseSessionService`** / **`VertexAiSessionService`**: Required for
  `user:` keys to actually persist across server restarts. `InMemorySessionService`
  loses all state when the process exits.
- **`app:` prefix**: Application-global state. Useful for shared configuration
  or feature flags but not user-specific long-term memory.
- **External vector store integration**: For semantic retrieval of long-term
  memories, store embeddings in a vector database (Vertex AI Vector Search,
  Pinecone, ChromaDB) and retrieve relevant entries by similarity.
- **ADK Agent Execution Lifecycle**: `user:` state is loaded at session start
  from the session service backend and is available throughout the session.

## Capabilities

- Stores user preferences and profile data persistently across sessions.
- Reads long-term user context from `user:` namespaced state.
- Accumulates historical interaction data across sessions.
- Integrates with vector stores for semantic long-term memory retrieval.
- Builds and updates user profiles incrementally.
- Injects relevant long-term context into agent instructions.

## Step-by-Step Instructions

1. **Identify the data** that must persist across sessions (user preferences,
   profile, history summaries).

2. **Write to `user:` prefixed state** in a tool function:
   ```python
   tool_context.state["user:preferred_language"] = "English"
   tool_context.state["user:preferred_seat"] = "aisle"
   tool_context.state["user:total_bookings"] = int(tool_context.state.get("user:total_bookings", 0)) + 1
   ```

3. **Read `user:` state** across sessions:
   ```python
   preferred_lang = tool_context.state.get("user:preferred_language", "English")
   ```

4. **Ensure a persistent session backend** is configured:
   ```python
   from google.adk.sessions import DatabaseSessionService
   session_service = DatabaseSessionService(db_url="postgresql://...")
   ```

5. **Inject long-term context into instruction** via `{user:key?}` placeholders:
   ```python
   instruction = "Respond in {user:preferred_language?}. User prefers {user:preferred_seat?} seats."
   ```

6. **Store summaries of past sessions** in `user:` state for compact long-term
   context (see `google-adk-memory-summarization` for compression):
   ```python
   tool_context.state["user:last_booking_summary"] = "JFK→LHR on 2024-03-15, BA178, confirmed"
   ```

7. **Retrieve semantically relevant memories** from a vector store if available
   (see `google-adk-vector-memory` and `google-adk-retrieval-augmented-generation`).

## Input Format

```python
{
  "session_id": "sess_new_456",
  "user_id": "user_42",
  "memory_type": "long_term",
  "content": "user prefers aisle seats and vegetarian meals",
  "metadata": {"source": "booking_tool", "updated_at": 1709900000},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "user:42:preferred_seat",
  "memory_retrieved": [
    {"key": "user:preferred_seat", "value": "aisle"},
    {"key": "user:preferred_meal", "value": "vegetarian"}
  ],
  "memory_injected": True
}
```

## Error Handling

- **`user:` state not persisting across sessions**: Check that `DatabaseSessionService`
  or `VertexAiSessionService` is used. `InMemorySessionService` does not persist `user:` state.
- **Missing user_id** (`tool_context.user_id` is empty): Always require user_id
  at session creation. Reject anonymous sessions if long-term memory is required.
- **`user:` key value type drift** (stored as str, expected int): Validate and
  cast on read: `int(state.get("user:total_bookings", 0))`.
- **`user:` state too large** (exceeds backend limits): Summarize and compress
  old entries. Store only the top-N most recent or relevant memories.
- **Race condition** (two sessions for the same user writing simultaneously):
  Use atomic increment operations where available in the session backend.

## Examples

### Example 1 — User preference storage and retrieval

```python
import time
from google.adk.tools import ToolContext

def save_user_preference(preference_key: str, preference_value: str, tool_context: ToolContext) -> dict:
    """Stores a user preference persistently across sessions.

    Args:
        preference_key (str): Preference name (e.g., 'preferred_seat', 'preferred_language').
        preference_value (str): Preference value.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'saved' (bool), 'key' (str), 'value' (str).
    """
    state_key = f"user:{preference_key}"
    tool_context.state[state_key] = preference_value
    tool_context.state[f"user:{preference_key}_updated_at"] = time.time()
    return {"saved": True, "key": state_key, "value": preference_value}


def get_user_preferences(tool_context: ToolContext) -> dict:
    """Retrieves the user's stored preferences from long-term memory.

    Args:
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'preferences' (dict) with all user: prefixed state values.
    """
    prefs = {
        k.replace("user:", ""): v
        for k, v in tool_context.state.items()
        if k.startswith("user:") and not k.endswith("_updated_at")
    }
    return {"preferences": prefs, "user_id": tool_context.user_id}
```

### Example 2 — Incrementing cross-session counters

```python
from google.adk.tools import ToolContext

def record_completed_booking(booking_reference: str, route: str, tool_context: ToolContext) -> dict:
    """Records a completed booking in long-term user history.

    Args:
        booking_reference (str): Booking reference code.
        route (str): Route flown, e.g., 'JFK-LHR'.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'total_bookings' (int).
    """
    total = int(tool_context.state.get("user:total_bookings", 0)) + 1
    tool_context.state["user:total_bookings"] = total
    tool_context.state["user:last_booking_ref"] = booking_reference
    tool_context.state["user:last_route"] = route
    return {"total_bookings": total, "booking_reference": booking_reference}
```

### Example 3 — Injecting long-term context into instruction

```python
from google.adk.agents import LlmAgent
from google.adk.agents.readonly_context import ReadonlyContext

def personalized_instruction(ctx: ReadonlyContext) -> str:
    """Builds a personalized instruction using long-term user memory."""
    name = ctx.state.get("user:name", "Valued Customer")
    lang = ctx.state.get("user:preferred_language", "English")
    seat = ctx.state.get("user:preferred_seat", "any")
    total = ctx.state.get("user:total_bookings", 0)

    history_note = (
        f"This user has completed {total} bookings with us. "
        f"Last route: {ctx.state.get('user:last_route', 'unknown')}."
        if total > 0 else "This is a new customer."
    )

    return f"""
You are FlightBot assisting {name}. Respond in {lang}.
{history_note}
Preferred seat type: {seat}. Proactively offer {seat} seats when available.
"""

agent = LlmAgent(
    name="personalized_agent",
    model="gemini-2.0-flash",
    instruction=personalized_instruction,
    tools=[search_flights, process_payment],
)
```

## Edge Cases

- **New user with no `user:` state**: All reads return None or defaults. Design
  onboarding tools to ask for and store initial preferences.
- **User changes preference mid-conversation**: Write the new value to `user:` state
  immediately. The update takes effect on the next turn.
- **`user:` state shared across concurrent sessions** (same user, two browser tabs):
  The last write wins. Avoid concurrent writes to the same `user:` key from parallel sessions.
- **GDPR/CCPA data deletion**: Implement a `delete_user_memory` tool that clears
  all `user:` state keys for a given user_id from the session backend.
- **Cross-tenant isolation**: `user:` state is scoped by `user_id` AND `app_name`.
  The same user_id in different apps does not share state.

## Integration Notes

- **`google-adk-memory-persistence`**: Covers the session service backend setup
  required for `user:` state to persist across server restarts and sessions.
- **`google-adk-vector-memory`**: For semantic retrieval of long-term memories
  (find relevant past interactions), integrate a vector store alongside `user:` state.
- **`google-adk-memory-summarization`**: Summarize old session content before
  storing in `user:` state to keep long-term memory compact.
- **`google-adk-context-injection`**: `{user:key?}` syntax injects long-term
  memory directly into agent instructions at zero code cost.
