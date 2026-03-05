---
name: sessions-state-memory
description: >
  Comprehensive reference for managing conversational context in Google Agent Development Kit (ADK)
  Python. Covers Session lifecycle (`Session`, `SessionService`, `InMemorySessionService`,
  `DatabaseSessionService`, `VertexAiSessionService`), State management (`session.state`, state
  prefixes `app:`, `user:`, `temp:`, `output_key`, `EventActions.state_delta`, `ToolContext.state`,
  `CallbackContext.state`, `{key}` instruction templating), and Memory (`MemoryService`,
  `InMemoryMemoryService`, `VertexAiMemoryBankService`, `add_session_to_memory`, `search_memory`,
  `PreloadMemoryTool`, `load_memory`). Consult when building any multi-turn agent, implementing
  cross-session recall, choosing a persistence backend, reading/writing state in tools or callbacks,
  or injecting state into agent instructions.
metadata:
  author: adk-skills
  version: "1.0"
---

# Sessions, State & Memory — Detailed Reference

## Overview

This skill covers the three pillars of conversational context in Google ADK (Python): **Session**, **State**, and **Memory**. Together they enable agents to track individual conversations, persist dynamic data within and across turns, and recall information from past interactions.

- **Session** is the container for a single conversation thread — it holds identification, event history, state, and timestamps. Managed by a `SessionService`.
- **State** (`session.state`) is the key-value scratchpad attached to each session. State prefixes (`app:`, `user:`, `temp:`, or no prefix) control scope and persistence behavior.
- **Memory** (`MemoryService`) is the long-term knowledge store that spans multiple past sessions, enabling agents to recall information beyond the current conversation.

These components sit in the **session layer** of the ADK architecture, between the runtime/Runner layer above and the storage backends below. The Runner orchestrates how sessions are created, how events (including state deltas) are appended, and how memory is ingested and queried.

For artifact storage (binary files, images, large blobs), see `artifact-management.md`.
For the Runner event loop and invocation lifecycle, see `runtime-and-runner.md`.

---

## Prerequisites

- Python 3.10+
- `pip install google-adk`
- For `DatabaseSessionService` with SQLite: `pip install aiosqlite`
- For `DatabaseSessionService` with PostgreSQL: `pip install asyncpg`
- For `DatabaseSessionService` with MySQL: `pip install aiomysql`
- For `VertexAiSessionService` or `VertexAiMemoryBankService`: `pip install google-adk[vertexai]`, a GCP project with Vertex AI API enabled, an Agent Engine resource, and `GOOGLE_CLOUD_PROJECT` / `GOOGLE_CLOUD_LOCATION` environment variables set
- Authentication for GCP services: `gcloud auth application-default login`

---

## Core Concept

### The Three Layers of Conversational Context

```
┌─────────────────────────────────────────────────────┐
│                    MEMORY                            │
│   Long-term knowledge store (cross-session)          │
│   MemoryService → add_session_to_memory /            │
│                   search_memory                      │
├─────────────────────────────────────────────────────┤
│                    STATE                             │
│   Key-value scratchpad within a session              │
│   Prefixes: app: | user: | temp: | (none)            │
│   Updated via: output_key, state_delta,              │
│     ToolContext.state, CallbackContext.state          │
├─────────────────────────────────────────────────────┤
│                   SESSION                            │
│   Single conversation thread container               │
│   Properties: id, app_name, user_id, events, state,  │
│               last_update_time                       │
│   Managed by: SessionService (create/get/list/delete)│
└─────────────────────────────────────────────────────┘
```

**Session** and **State** focus on the current interaction — the history and data of one active conversation. They are managed by a `SessionService`.

**Memory** focuses on the past — a searchable archive potentially spanning many conversations. It is managed by a `MemoryService`.

### How It Connects to the Runner Event Loop

Every interaction flows through the Runner:

1. The Runner retrieves (or creates) a `Session` from the `SessionService`.
2. The agent processes the user message, reading `session.state` and `session.events` for context.
3. The agent generates a response. Any state changes are packaged as `EventActions.state_delta` inside an `Event`.
4. The Runner calls `session_service.append_event(session, event)`, which persists the event, applies state deltas, and updates `last_update_time`.
5. For memory: at an appropriate point (e.g., session end), the application calls `memory_service.add_session_to_memory(session)` to ingest session data into long-term storage.

State changes made via `ToolContext.state`, `CallbackContext.state`, or `output_key` are automatically tracked by the framework — they populate the `state_delta` field of the corresponding `EventActions`, which the `SessionService` applies when the event is appended.

---

## When to Use

- When building any multi-turn conversational agent that must maintain context across turns within a conversation (Session + State).
- When a `SequentialAgent` pipeline has intermediate outputs that a downstream `LlmAgent` must read via `{key}` template substitution in its `instruction` (State).
- When you need user preferences to persist across all sessions for the same user (State with `user:` prefix + persistent `SessionService`).
- When application-wide configuration must be accessible to all agents across all users (State with `app:` prefix).
- When tools need to store intermediate computation results that should not survive past the current invocation (State with `temp:` prefix).
- When an agent must recall information from prior conversations — e.g., "What did we discuss about project X last week?" (Memory).
- When choosing between in-memory (development), database (self-managed production), or Vertex AI (managed production) persistence backends.
- When you need to update session state from outside the Runner loop using manual `EventActions.state_delta` and `append_event`.

---

## Rules

- **State must be updated through the event system.** Always use `output_key`, `EventActions.state_delta`, `ToolContext.state`, or `CallbackContext.state`. Never modify `session.state` directly on a `Session` object retrieved from the `SessionService` — this bypasses event tracking, breaks persistence, and is not thread-safe.
- **State keys are always strings.** Values must be serializable (strings, numbers, booleans, simple lists/dicts of these types). Do not store custom class instances, functions, or connection objects.
- **State prefixes determine scope and persistence:**
  - No prefix → session-scoped (current session only).
  - `user:` → user-scoped (shared across all sessions for the same `user_id` within the same `app_name`).
  - `app:` → app-scoped (shared across all users and sessions for the same `app_name`).
  - `temp:` → invocation-scoped (discarded after the current invocation completes; does not carry to the next turn).
- **`user:` and `app:` prefixes only provide cross-session persistence with `DatabaseSessionService` or `VertexAiSessionService`.** With `InMemorySessionService`, they are stored but lost on restart.
- **`output_key` only works on `LlmAgent`.** It captures the agent's final text response and writes it to `session.state[output_key]`. It does NOT capture tool results or intermediate outputs.
- **`{key}` templating in `LlmAgent.instruction` requires the key to exist in `session.state`.** If the key is missing, an error is raised. Use `{key?}` for optional keys that may not be present.
- **`InMemorySessionService` and `InMemoryMemoryService` lose all data on application restart.** Use them only for development and testing.
- **`DatabaseSessionService` requires an async database driver** (e.g., `sqlite+aiosqlite`, `asyncpg`, `aiomysql`).
- **A single process can only be configured with one `MemoryService` backend.** The framework does not natively support multiple memory services simultaneously, though you can build a custom composite service.
- **`add_session_to_memory` should be called after a session is considered complete** or has yielded significant information. Calling it on every turn would create redundant entries.
- **The `PreloadMemoryTool` runs a memory search automatically at the beginning of each turn.** The `load_memory` tool lets the agent decide when to search. Choose based on whether memory retrieval should be automatic or on-demand.

---

## Example

### Complete Example: Session Lifecycle, State Updates, and Memory Recall

```python
import asyncio
from google.adk.agents import LlmAgent
from google.adk.sessions import InMemorySessionService
from google.adk.memory import InMemoryMemoryService
from google.adk.runners import Runner
from google.adk.tools import load_memory
from google.adk.tools import ToolContext
from google.genai.types import Content, Part

# --- Constants ---
APP_NAME = "session_state_memory_demo"
USER_ID = "demo_user"
MODEL = "gemini-2.0-flash"

# --- Tool that writes state ---
def track_preference(category: str, value: str, tool_context: ToolContext) -> dict:
    """Records a user preference in session state.

    Args:
        category: The preference category (e.g., 'color', 'language').
        value: The preference value.
        tool_context: Provided by ADK framework.

    Returns:
        Confirmation of the stored preference.
    """
    # Writing via tool_context.state is automatically tracked in state_delta
    tool_context.state[f"user:pref_{category}"] = value
    return {"status": "stored", "category": category, "value": value}

# --- Agent that captures preferences ---
capture_agent = LlmAgent(
    model=MODEL,
    name="PreferenceCapture",
    instruction=(
        "You help users set preferences. Use the track_preference tool "
        "to store any preference the user mentions."
    ),
    tools=[track_preference],
    output_key="last_capture_response",  # Saves final text to state
)

# --- Agent that recalls from memory ---
recall_agent = LlmAgent(
    model=MODEL,
    name="MemoryRecall",
    instruction=(
        "Answer the user's question. Use the 'load_memory' tool "
        "if the answer might be in past conversations."
    ),
    tools=[load_memory],
)

# --- Services ---
session_service = InMemorySessionService()
memory_service = InMemoryMemoryService()


async def main():
    # === SESSION 1: Capture preferences ===
    runner1 = Runner(
        agent=capture_agent,
        app_name=APP_NAME,
        session_service=session_service,
        memory_service=memory_service,
    )

    session1 = await session_service.create_session(
        app_name=APP_NAME,
        user_id=USER_ID,
        session_id="session_capture",
        state={"user:name": "Bhavya"},  # Initialize state at creation
    )
    print(f"Session 1 created. Initial state: {session1.state}")

    # Send a message
    msg = Content(parts=[Part(text="My favorite color is teal.")], role="user")
    async for event in runner1.run_async(
        user_id=USER_ID, session_id="session_capture", new_message=msg
    ):
        if event.is_final_response() and event.content and event.content.parts:
            print(f"Agent: {event.content.parts[0].text}")

    # Check state after the interaction
    updated = await session_service.get_session(
        app_name=APP_NAME, user_id=USER_ID, session_id="session_capture"
    )
    print(f"State after turn: {updated.state}")
    # Expected: includes 'user:pref_color': 'teal', 'last_capture_response': '...',
    #           and 'user:name': 'Bhavya'

    # === INGEST SESSION INTO MEMORY ===
    await memory_service.add_session_to_memory(updated)
    print("Session 1 ingested into memory.")

    # === SESSION 2: Recall from memory ===
    runner2 = Runner(
        agent=recall_agent,
        app_name=APP_NAME,
        session_service=session_service,
        memory_service=memory_service,
    )

    await session_service.create_session(
        app_name=APP_NAME,
        user_id=USER_ID,
        session_id="session_recall",
    )

    msg2 = Content(
        parts=[Part(text="What is my favorite color?")], role="user"
    )
    async for event in runner2.run_async(
        user_id=USER_ID, session_id="session_recall", new_message=msg2
    ):
        if event.is_final_response() and event.content and event.content.parts:
            print(f"Recall Agent: {event.content.parts[0].text}")

    # === LIST AND DELETE SESSIONS ===
    all_sessions = await session_service.list_sessions(
        app_name=APP_NAME, user_id=USER_ID
    )
    print(f"Active sessions: {[s.id for s in all_sessions.sessions]}")

    # Cleanup
    await session_service.delete_session(
        app_name=APP_NAME, user_id=USER_ID, session_id="session_capture"
    )
    await session_service.delete_session(
        app_name=APP_NAME, user_id=USER_ID, session_id="session_recall"
    )
    print("Sessions deleted.")


# asyncio.run(main())
```

---

## Decision Table

| Need | Recommended Component | Why |
|---|---|---|
| Track conversation history within one chat | `Session` + `SessionService` | Session holds chronological events and state for a single thread |
| Store a value that persists only during this conversation | `session.state['my_key']` (no prefix) | Session-scoped, cleared when session is deleted |
| Store a user preference across all conversations | `session.state['user:pref']` + persistent `SessionService` | `user:` prefix shares state across sessions for the same user |
| Store an application-wide setting | `session.state['app:setting']` + persistent `SessionService` | `app:` prefix shares across all users and sessions |
| Pass intermediate data between tool calls within one turn | `session.state['temp:data']` | `temp:` prefix is discarded after the invocation completes |
| Save an agent's final text response to state | `output_key` on `LlmAgent` | Automatically writes the response string to the named state key |
| Update state from a tool function | `tool_context.state['key'] = value` | Changes are automatically tracked in `EventActions.state_delta` |
| Update state from a callback | `callback_context.state['key'] = value` | Same tracking mechanism as `ToolContext` |
| Update state from outside the Runner | `EventActions(state_delta={...})` + `session_service.append_event(...)` | Manual event creation for external state updates |
| Inject state into agent instructions | `{key}` templating in `LlmAgent.instruction` | Framework replaces placeholders before sending to LLM |
| Recall information from past sessions | `MemoryService` + `load_memory` or `PreloadMemoryTool` | Long-term knowledge store with search capability |
| Quick local development | `InMemorySessionService` + `InMemoryMemoryService` | No setup, no persistence, no dependencies |
| Self-managed production persistence | `DatabaseSessionService` + SQL database | Persistent, you control the infrastructure |
| Managed production persistence | `VertexAiSessionService` + `VertexAiMemoryBankService` | Fully managed by Google Cloud with semantic search |

---

## Common Patterns

### Pattern 1: State-Driven Dynamic Instructions

Use `{key}` templating to inject session state values into an `LlmAgent`'s instruction, making behavior dynamic without relying on natural language directives.

```python
from google.adk.agents import LlmAgent

agent = LlmAgent(
    name="DynamicGreeter",
    model="gemini-2.0-flash",
    instruction=(
        "You are a {user:role} assistant for {user:name}. "
        "Respond in {user:preferred_language}. "
        "Current task: {current_task}."
    ),
)
# Requires session.state to contain:
#   'user:role', 'user:name', 'user:preferred_language', 'current_task'
# Use {key?} syntax for optional keys that may not exist.
```

### Pattern 2: InstructionProvider for Complex Templates

When instructions contain literal curly braces (e.g., JSON examples), use an `InstructionProvider` function to prevent the framework from interpreting them as state placeholders.

```python
from google.adk.agents import LlmAgent
from google.adk.agents.readonly_context import ReadonlyContext
from google.adk.utils import instructions_utils

async def build_instruction(context: ReadonlyContext) -> str:
    user_name = context.state.get("user:name", "User")
    template = (
        f"Hello {user_name}. "
        'Format output as JSON: {"result": "<value>", "confidence": <number>}.'
    )
    # inject_session_state replaces valid {key} placeholders
    # but leaves JSON braces alone (not valid identifiers)
    return await instructions_utils.inject_session_state(template, context)

agent = LlmAgent(
    name="JsonHelper",
    model="gemini-2.0-flash",
    instruction=build_instruction,
)
```

### Pattern 3: State Updates via Manual Event Injection

Update session state from outside the Runner loop — useful for system-level state changes, background processes, or admin overrides.

```python
import time
from google.adk.events import Event, EventActions

async def update_state_externally(session_service, session, changes: dict):
    """Update session state by appending a system event."""
    actions = EventActions(state_delta=changes)
    system_event = Event(
        invocation_id="external_update",
        author="system",
        actions=actions,
        timestamp=time.time(),
    )
    await session_service.append_event(session, system_event)

# Usage:
# await update_state_externally(
#     session_service, session,
#     {"user:tier": "premium", "app:maintenance_mode": False}
# )
```

### Pattern 4: Auto-Save Sessions to Memory via Callback

Use an `after_agent_callback` to automatically ingest every completed conversation turn into long-term memory.

```python
from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.adk.tools.preload_memory_tool import PreloadMemoryTool

async def save_to_memory(callback_context: CallbackContext):
    """After each agent turn, save the session to memory."""
    memory_service = callback_context._invocation_context.memory_service
    if memory_service:
        session = callback_context._invocation_context.session
        await memory_service.add_session_to_memory(session)
    return None  # Do not override the agent's response

agent = LlmAgent(
    model="gemini-2.0-flash",
    name="MemoryAwareAgent",
    instruction="Answer the user's questions helpfully.",
    tools=[PreloadMemoryTool()],  # Auto-retrieve memories at turn start
    after_agent_callback=save_to_memory,
)
```

### Pattern 5: DatabaseSessionService for Production Persistence

Configure a persistent session backend using a relational database.

```python
from google.adk.sessions import DatabaseSessionService
from google.adk.runners import Runner

# SQLite (async driver required)
db_url = "sqlite+aiosqlite:///./agent_sessions.db"
session_service = DatabaseSessionService(db_url=db_url)

# PostgreSQL
# db_url = "postgresql+asyncpg://user:pass@host:5432/dbname"
# session_service = DatabaseSessionService(db_url=db_url)

runner = Runner(
    agent=my_agent,
    app_name="production_app",
    session_service=session_service,
)

# With DatabaseSessionService:
# - Session-scoped state (no prefix) persists across restarts
# - user: prefixed state is shared across sessions for the same user
# - app: prefixed state is shared across all users
# - temp: prefixed state is still invocation-only (never persisted)
```

---

## Edge Cases

### Missing State Key in Instruction Template

If an `LlmAgent.instruction` contains `{key}` but that key does not exist in `session.state`, the framework raises an error at runtime. Use `{key?}` to make the substitution optional — if the key is absent, the placeholder is replaced with an empty string. Always initialize required state keys when creating the session or via a `before_agent_callback`.

### Direct State Mutation on Session Object

Modifying `session.state['key'] = value` directly on a `Session` object retrieved from `session_service.get_session(...)` bypasses event tracking entirely. The change will not be recorded in event history, will not be persisted by `DatabaseSessionService` or `VertexAiSessionService`, is not thread-safe, and does not update `last_update_time`. Always use `ToolContext.state`, `CallbackContext.state`, `output_key`, or `EventActions.state_delta` via `append_event`.

### State Key Collisions Across Agents

In multi-agent systems (e.g., `SequentialAgent` pipelines), multiple agents share the same `session.state`. If two agents or tools write to the same key without coordination, the last write wins and earlier values are silently overwritten. Mitigate by using descriptive, namespaced keys (e.g., `agent_name:key`) and documenting which agent owns which keys.

### temp: State Does Not Survive Across Turns

The `temp:` prefix means invocation-scoped — the data is discarded after the Runner completes the current invocation (one user message → one final response cycle). If you store data as `temp:result` in turn 1, it will not be available in turn 2. Use session-scoped (no prefix) or `user:` prefixed keys for data that must persist across turns.

### InMemorySessionService Data Loss

`InMemorySessionService` stores everything in process memory. If the application restarts, crashes, or scales to multiple instances, all session data is lost. In a multi-instance deployment, different instances cannot share sessions. Use `DatabaseSessionService` or `VertexAiSessionService` for any production deployment.

### InMemoryMemoryService Keyword Matching Limitations

`InMemoryMemoryService` performs basic keyword matching, not semantic search. It stores full conversation transcripts and matches on literal terms. For production use cases requiring nuanced recall (e.g., "What did we discuss about the Q3 budget?"), use `VertexAiMemoryBankService` which provides LLM-powered memory extraction and semantic search.

### Memory Ingestion Timing

Calling `add_session_to_memory` after every single turn creates redundant or partial entries. The recommended pattern is to call it when a session is considered complete (e.g., user says goodbye, task is finished) or at meaningful checkpoints. For the auto-save callback pattern, consider adding logic to only ingest when the session has meaningful new content.

### output_key Writes Only the Final Text Response

`output_key` on `LlmAgent` captures only the agent's final text response as a string. It does not capture tool call results, intermediate reasoning, structured output, or responses from sub-agents. If you need to store tool results or structured data, use `ToolContext.state` or `EventActions.state_delta` explicitly.

### DatabaseSessionService Schema Migration

The session database schema changed in ADK Python v1.22.0. If you are upgrading from an earlier version, you must migrate your database. See the official migration guide at `https://google.github.io/adk-docs/sessions/session/migrate/`.

---

## Failure Handling

### SessionService.get_session Returns None

If the session ID does not exist (was deleted or never created), `get_session` returns `None`. The Runner will fail if given a non-existent session ID. Always verify session existence or handle `None` before passing to the Runner.

### DatabaseSessionService Connection Failure

If the database is unreachable, `DatabaseSessionService` raises a connection error during `create_session`, `get_session`, or `append_event`. ADK does not retry automatically. Implement connection pooling and retry logic at the database driver level (e.g., using SQLAlchemy's pool configuration).

### MemoryService.search_memory Returns Empty Results

If no matching memories exist or the query terms don't match (especially with `InMemoryMemoryService`'s keyword matching), `search_memory` returns a `SearchMemoryResponse` with an empty list of `MemoryResult` objects. The agent's memory tool will report no results found. This is not an error — the agent should gracefully handle the absence of memory context.

### Non-Serializable State Values

Storing a non-serializable object (class instance, function, file handle) in state will cause `DatabaseSessionService` or `VertexAiSessionService` to fail during `append_event` when attempting to serialize the state delta. `InMemorySessionService` may accept it but the value will be lost if you ever switch to a persistent backend. Always store only JSON-serializable primitives.

---

## Scalability Notes

### SessionService Implementation Comparison

| Feature | `InMemorySessionService` | `DatabaseSessionService` | `VertexAiSessionService` |
|---|---|---|---|
| Persistence | None (process memory) | Yes (SQL database) | Yes (Vertex AI managed) |
| Multi-instance support | No (single process) | Yes (shared database) | Yes (cloud API) |
| Setup complexity | None | Moderate (database + async driver) | Low (GCP project + Agent Engine) |
| Scaling ceiling | Process memory limits | Database capacity | Google Cloud quotas |
| `user:` / `app:` cross-session | Stored but lost on restart | Fully persistent | Fully persistent |

### MemoryService Implementation Comparison

| Feature | `InMemoryMemoryService` | `VertexAiMemoryBankService` |
|---|---|---|
| Persistence | None (process memory) | Yes (Vertex AI managed) |
| Search type | Basic keyword matching | Semantic search (LLM-powered) |
| Memory extraction | Stores full conversation | Extracts meaningful information, consolidates with existing memories |
| Setup | None | Agent Engine in Vertex AI |

### Concurrency Constraints

- `InMemorySessionService` is not thread-safe across multiple `asyncio` tasks modifying the same session simultaneously. The Runner serializes event appending, but external state updates must be coordinated.
- `DatabaseSessionService` relies on database-level locking. Ensure your database supports concurrent writes (PostgreSQL recommended over SQLite for production).
- Vertex AI services are rate-limited by Google Cloud quotas. Check your project's Vertex AI API quotas for session and memory operations.

### Token Budget Implications

- Long sessions with many events increase the context window consumed by `session.events`. Consider using context compression (see `context-compression.md`) for sessions with deep history.
- `PreloadMemoryTool` adds retrieved memory content to the LLM context at every turn, consuming tokens. Use `load_memory` instead if memory retrieval should be selective.

---

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
|---|---|---|
| Directly modifying `session.state` on a retrieved `Session` object | Bypasses event tracking, breaks persistence, not thread-safe | Use `ToolContext.state`, `CallbackContext.state`, `output_key`, or `EventActions.state_delta` |
| Storing complex objects (class instances, functions) in state | Serialization failure with persistent backends | Store only JSON-serializable primitives (str, int, float, bool, list, dict) |
| Using `InMemorySessionService` in production | Data loss on restart, no multi-instance support | Use `DatabaseSessionService` or `VertexAiSessionService` |
| Using `temp:` prefix for data needed across turns | Data is discarded after each invocation | Use session-scoped (no prefix) or `user:` prefix |
| Calling `add_session_to_memory` on every single turn | Creates redundant/partial memory entries | Ingest at session completion or meaningful checkpoints |
| Using `output_key` to capture tool results | `output_key` only captures the agent's final text response | Use `ToolContext.state` to store tool results explicitly |
| Using identical state key names across agents without namespacing | Silent overwrites, unpredictable behavior | Namespace keys by agent name: `agent_name:key` |
| Relying on `InMemoryMemoryService` for complex recall | Keyword matching misses semantic relationships | Use `VertexAiMemoryBankService` for production |
| Referencing `{key}` in instruction without ensuring key exists in state | Runtime error | Use `{key?}` for optional keys, or initialize state at session creation |

---

## Guidelines

1. **Always use event-tracked mechanisms for state updates.** Write to `ToolContext.state` or `CallbackContext.state` inside tools and callbacks. These modifications are automatically included in `EventActions.state_delta` and persisted by the `SessionService` when the event is appended.

2. **Choose state key prefixes deliberately.** Use no prefix for session-local data, `user:` for cross-session user preferences, `app:` for global application settings, and `temp:` for single-invocation intermediate data. Document the prefix strategy for your project.

3. **Initialize required state keys at session creation.** Pass a `state` dictionary to `session_service.create_session(state={...})` to ensure all keys referenced by `{key}` instruction templates or tool logic exist from the first turn.

4. **Use `InstructionProvider` when instructions contain literal curly braces.** If your instruction includes JSON examples or template syntax, provide a callable instead of a string to prevent the framework from treating `{...}` as state placeholders.

5. **Select a persistent `SessionService` for production.** Use `DatabaseSessionService` with an async driver (e.g., `postgresql+asyncpg`) for self-managed deployments, or `VertexAiSessionService` for fully managed Google Cloud deployments. Reserve `InMemorySessionService` for local development only.

6. **Call `add_session_to_memory` at meaningful boundaries.** Ingest sessions into the `MemoryService` when conversations reach a logical conclusion or significant checkpoint — not on every turn. Use an `after_agent_callback` to automate this with appropriate gating logic.

7. **Choose between `PreloadMemoryTool` and `load_memory` based on retrieval strategy.** Use `PreloadMemoryTool` when every turn should have memory context automatically. Use `load_memory` when the agent should decide when memory retrieval is needed, saving tokens on turns that don't require it.

8. **Migrate the session database schema when upgrading ADK past v1.22.0.** The `DatabaseSessionService` schema changed in this version. Follow the official migration guide to avoid data loss or runtime errors with existing databases.

---

## References

- [Official ADK Docs — Sessions & Memory Introduction](https://google.github.io/adk-docs/sessions/)
- [Official ADK Docs — Session](https://google.github.io/adk-docs/sessions/session/)
- [Official ADK Docs — State](https://google.github.io/adk-docs/sessions/state/)
- [Official ADK Docs — Memory](https://google.github.io/adk-docs/sessions/memory/)
- [Official ADK Docs — Context](https://google.github.io/adk-docs/context/)
- [Official ADK Docs — Events](https://google.github.io/adk-docs/events/)
- [ADK Python API Reference](https://google.github.io/adk-docs/api-reference/python/)
- [ADK Python GitHub](https://github.com/google/adk-python)
