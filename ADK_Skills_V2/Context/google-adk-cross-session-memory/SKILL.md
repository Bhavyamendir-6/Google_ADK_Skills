---
name: google-adk-cross-session-memory
description: >
  Use this skill when a Google ADK agent needs to persist and retrieve
  information across multiple separate sessions for the same user. Covers
  the user: prefix in session.state, ADK MemoryService, and external memory
  store integration to enable agents to remember facts, preferences, and
  history beyond the lifespan of a single session.
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

This skill enables Google ADK agents to maintain a persistent memory of user
interactions, preferences, and facts across distinct sessions. Using the `user:`
state prefix and ADK's `MemoryService`, this skill ensures that when a user
returns in a new session, the agent retains what it learned in previous ones —
without relying on the LLM to infer context from scratch.

## When to Use

- When user facts, preferences, or decisions from a previous session should
  inform the current session's behavior.
- When building agents that must know "who this user is" across conversations.
- When implementing long-term user personalization (e.g., language preference,
  dietary restrictions, past order history).
- When using `VertexAiMemoryBankService` or a custom `MemoryService` for
  production-grade memory persistence.

## When NOT to Use

- Do not use `user:` prefix state for session-only data — use no prefix instead.
- Do not store sensitive PII in `user:` state without encryption and compliance review.
- Do not use `InMemorySessionService` for cross-session memory — it is lost on
  restart. Use `DatabaseSessionService` or `VertexAiSessionService`.

## Google ADK Context

- **`user:` prefix**: State keys prefixed with `user:` are scoped to the `user_id`
  and persisted across all sessions for that user (with persistent `SessionService`).
  ADK merges `user:` state into `session.state` on `get_session()`.
- **`app:` prefix**: State keys prefixed with `app:` are shared across all users
  and sessions. Use for global cross-session facts.
- **`tool_context.user_id`**: The current user's ID. Used to query external
  memory stores by user.
- **`MemoryService`**: ADK interface for storing and searching structured memory
  entries. Implementations:
  - `InMemoryMemoryService`: Non-persistent, for dev/testing.
  - `VertexAiMemoryBankService`: Google Cloud production memory service.
- **`add_session_to_memory()`**: Adds a completed `Session` to the `MemoryService`
  for later retrieval via semantic search.
- **`search_memory()`**: Retrieves relevant memory entries for the current query.
- **ADK Execution Lifecycle**: Cross-session memory is loaded at session start
  (via `search_memory`) and updated at session end (via `add_session_to_memory`).

## Capabilities

- Persists user-specific data across sessions via `user:` prefix state keys.
- Loads cross-session memory at session start using `MemoryService.search_memory()`.
- Stores completed sessions into memory via `MemoryService.add_session_to_memory()`.
- Reads user memories in tool functions via `tool_context.user_id`.
- Injects retrieved memories into agent instructions.
- Supports semantic memory search (Vertex AI Memory Bank).

## Step-by-Step Instructions

1. **Initialize the `MemoryService`** alongside the `SessionService`:
   ```python
   from google.adk.memory import VertexAiMemoryBankService
   memory_service = VertexAiMemoryBankService(project="my-project", location="us-central1")
   ```

2. **At session start, search for relevant memories**:
   ```python
   memories = await memory_service.search_memory(
       app_name="my_app",
       user_id=user_id,
       query=user_message_text,
   )
   ```

3. **Inject retrieved memories into session state**:
   ```python
   memory_text = "\n".join(m.content for m in memories.memories)
   session = await session_service.create_session(
       app_name="my_app",
       user_id=user_id,
       state={"user:past_context": memory_text, "user:name": "Alice"},
   )
   ```

4. **Within tools, write user-scoped state** using `user:` prefix:
   ```python
   tool_context.state["user:preferred_language"] = "French"
   tool_context.state["user:last_order_id"] = "ORD-9821"
   ```

5. **At session end, persist the session to memory**:
   ```python
   session = await session_service.get_session(
       app_name="my_app", user_id=user_id, session_id=session_id
   )
   await memory_service.add_session_to_memory(session)
   ```

6. **Inject cross-session memory into agent instructions**:
   ```python
   LlmAgent(instruction="User context from past sessions: {user:past_context?}.\nAssist: {user:name?}.")
   ```

7. **Validate `user:` state** on retrieval — it may be stale. Check timestamps
   stored alongside values if freshness matters.

## Input Format

```python
{
  "session_id": "sess_new_456",
  "user_id": "user_xyz",
  "state": {
    "user:name": "Alice",
    "user:preferred_language": "English",
    "user:past_context": "Previously booked NYC to London flight.",
    "user:last_session_id": "sess_abc123"
  },
  "metadata": {
    "memory_search_query": "user preferences and past orders"
  }
}
```

## Output Format

```python
{
  "state_updated": True,
  "state_key": "user:preferred_language",
  "state_value": "French",
  "scope": "user",
  "persisted_to_memory": True
}
```

## Error Handling

- **`VertexAiMemoryBankService` unavailable**: Fall back to reading `user:`
  state from `DatabaseSessionService`. Log the Memory Bank failure.
- **`search_memory()` returns empty results**: Proceed with an empty
  `user:past_context`. Do not block session creation.
- **`user:` key missing in new session**: Default to `""` or `None`. The agent
  must handle first-time users gracefully.
- **Stale `user:` state** (e.g., preference changed): Implement a `user:updated_at`
  timestamp key. Update it whenever a `user:` key is changed.
- **Memory serialization error**: Only store JSON-serializable values. Use
  `json.dumps()` for complex structures stored as strings.
- **`add_session_to_memory()` fails**: Log the error and retry asynchronously.
  Do not block the session cleanup flow.

## Examples

### Example 1 — Cross-session memory with VertexAiMemoryBankService

```python
import asyncio
from google.adk.sessions import VertexAiSessionService
from google.adk.memory import VertexAiMemoryBankService
from google.adk.runners import Runner
from google.adk.agents import LlmAgent
from google.genai.types import Content, Part

async def run_with_memory(user_id: str, user_message: str):
    PROJECT = "my-project"
    LOCATION = "us-central1"
    REASONING_ENGINE_ID = "projects/my-project/locations/us-central1/reasoningEngines/123"

    session_service = VertexAiSessionService(project=PROJECT, location=LOCATION)
    memory_service = VertexAiMemoryBankService(project=PROJECT, location=LOCATION)

    # Search past memories
    memories = await memory_service.search_memory(
        app_name=REASONING_ENGINE_ID, user_id=user_id, query=user_message
    )
    past_context = "\n".join(m.content for m in memories.memories) if memories.memories else ""

    # Create new session with memory-loaded state
    session = await session_service.create_session(
        app_name=REASONING_ENGINE_ID,
        user_id=user_id,
        state={"user:past_context": past_context},
    )

    agent = LlmAgent(
        name="assistant",
        model="gemini-2.0-flash",
        instruction="Previous interactions: {user:past_context?}. Now assist the user.",
    )
    runner = Runner(agent=agent, app_name=REASONING_ENGINE_ID, session_service=session_service)

    msg = Content(parts=[Part(text=user_message)])
    for event in runner.run(user_id=user_id, session_id=session.id, new_message=msg):
        if event.is_final_response():
            print("Agent:", event.content)

    # Persist session to memory for next session
    completed = await session_service.get_session(
        app_name=REASONING_ENGINE_ID, user_id=user_id, session_id=session.id
    )
    await memory_service.add_session_to_memory(completed)
```

### Example 2 — Writing user-scoped state in a tool

```python
from google.adk.tools import ToolContext

def save_user_preference(language: str, theme: str, tool_context: ToolContext) -> dict:
    """Saves user preferences to cross-session state.

    Args:
        language (str): Preferred language code, e.g. 'fr'.
        theme (str): UI theme preference: 'light' or 'dark'.
        tool_context (ToolContext): ADK tool context.

    Returns:
        dict: Contains 'saved' (bool).
    """
    import time
    tool_context.state["user:preferred_language"] = language
    tool_context.state["user:theme"] = theme
    tool_context.state["user:preferences_updated_at"] = time.time()
    return {"saved": True}
```

## Edge Cases

- **First-time user (no memory)**: `search_memory()` returns empty list.
  Initialize `user:past_context` to `""`. No error should occur.
- **User ID changes between sessions**: Memory is keyed by `user_id`. If the
  ID changes (e.g., anonymous → authenticated), migrate `user:` state explicitly.
- **Memory Bank quota exceeded**: Check for quota errors. Implement a local
  `DatabaseSessionService` fallback for `user:` state.
- **Conflicting `user:` state across devices**: ADK does not implement conflict
  resolution — last write wins. Design for idempotent `user:` state writes.

## Integration Notes

- **`VertexAiSessionService`**: Required for `user:` state persistence in GCP.
  `InMemorySessionService` loses `user:` state on restart.
- **`google-adk-user-specific-context`**: Extends this skill with per-user
  context injection and personalization patterns.
- **`google-adk-context-summarization`**: Store the final conversation summary
  in `user:last_summary` for retrieval in subsequent sessions.
- **ADK `MemoryService`**: Provides semantic search over past sessions.
  Pair with `google-adk-session-management` for lifecycle control.
