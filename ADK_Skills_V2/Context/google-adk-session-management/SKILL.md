---
name: google-adk-session-management
description: >
  Use this skill when a Google ADK agent needs to create, retrieve, resume,
  list, or delete sessions via SessionService. Covers the full session lifecycle
  using InMemorySessionService, DatabaseSessionService, and VertexAiSessionService,
  including session initialization with state, session resumption across turns,
  and safe session cleanup.
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

This skill governs the complete lifecycle of ADK `Session` objects — from
creation through resumption to deletion. It ensures agents can correctly
initialize sessions with seed state, resume existing conversations across
requests, and clean up sessions when complete. It also covers selecting the
correct `SessionService` implementation for the deployment environment.

## When to Use

- When initializing the agent runner and session service for a new application.
- When a new user interaction begins and a session must be created or resumed.
- When migrating from `InMemorySessionService` to a persistent backend.
- When implementing session expiry, cleanup, or listing functionality.
- When the agent must resume a previous conversation from a known `session_id`.

## When NOT to Use

- Do not use to manage state within an active session — use
  `google-adk-conversation-state-tracking` for that.
- Do not use `InMemorySessionService` in production — it loses all data on restart.
- Do not modify `Session` properties directly — always use `SessionService` methods.

## Google ADK Context

- **`SessionService`**: Central manager for session lifecycle. Three implementations:
  - `InMemorySessionService`: Non-persistent. For development/testing only.
  - `DatabaseSessionService`: Persistent via relational DB (SQLite, PostgreSQL, MySQL).
  - `VertexAiSessionService`: Persistent via Google Cloud Vertex AI Agent Engine.
- **`Session` object** properties:
  - `session.id`: Unique session identifier string.
  - `session.app_name`: Application namespace.
  - `session.user_id`: User associated with this session.
  - `session.state`: Dict of current session state (read-only outside events).
  - `session.events`: Ordered list of `Event` objects (full conversation history).
  - `session.last_update_time`: Unix timestamp of last event.
- **`tool_context.session_id`**: The active session ID accessible in tool functions.
- **ADK Execution Lifecycle**: The `Runner` automatically calls `get_session()` at
  the start of each invocation and `append_event()` after each turn.

## Capabilities

- Create new sessions with optional seed state.
- Resume existing sessions by `session_id`.
- List all sessions for a given `user_id` and `app_name`.
- Delete sessions when conversations are complete.
- Initialize the `Runner` with the correct `SessionService`.
- Migrate between session service backends without data loss (with export/import).
- Handle missing or expired sessions gracefully.

## Step-by-Step Instructions

1. **Select the correct `SessionService`** based on environment:
   - Local development → `InMemorySessionService`
   - Self-hosted production → `DatabaseSessionService`
   - GCP production → `VertexAiSessionService`

2. **Initialize the `SessionService`**:
   ```python
   from google.adk.sessions import DatabaseSessionService
   session_service = DatabaseSessionService(db_url="sqlite+aiosqlite:///agent.db")
   ```

3. **Create a new session** with optional initial state:
   ```python
   session = await session_service.create_session(
       app_name="my_app",
       user_id="user_123",
       session_id="sess_abc",  # Optional; auto-generated if omitted
       state={"onboarding_complete": False, "user:theme": "dark"},
   )
   ```

4. **Initialize the Runner** with the agent and session service:
   ```python
   from google.adk.runners import Runner
   runner = Runner(agent=my_agent, app_name="my_app", session_service=session_service)
   ```

5. **Resume an existing session** by passing its `session_id` to `runner.run()`:
   ```python
   for event in runner.run(user_id="user_123", session_id="sess_abc", new_message=msg):
       if event.is_final_response():
           print(event.content)
   ```

6. **Retrieve session state** after a run:
   ```python
   session = await session_service.get_session(
       app_name="my_app", user_id="user_123", session_id="sess_abc"
   )
   print(session.state)
   ```

7. **Delete a session** when the conversation ends:
   ```python
   await session_service.delete_session(
       app_name="my_app", user_id="user_123", session_id="sess_abc"
   )
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",        # Unique; auto-generated if omitted
  "user_id": "user_xyz",
  "app_name": "my_application",
  "state": {                          # Optional initial state
    "onboarding_complete": False,
    "user:theme": "dark"
  },
  "metadata": {}
}
```

## Output Format

Session creation/retrieval result:
```python
{
  "state_updated": True,
  "session_id": "sess_abc123",
  "user_id": "user_xyz",
  "app_name": "my_application",
  "last_update_time": 1700000000.0,
  "event_count": 4
}
```

## Error Handling

- **`get_session()` returns `None`**: Session does not exist or has been deleted.
  Check for `None` before using. Create a new session if appropriate.
- **Duplicate `session_id`**: `create_session()` with an existing ID may raise
  or return the existing session depending on the backend. Use unique IDs.
- **DB connection failure** (`DatabaseSessionService`): Wrap in try/except.
  Return a transient error and retry with exponential backoff.
- **`VertexAiSessionService` auth failure**: Ensure `GOOGLE_CLOUD_PROJECT` and
  `GOOGLE_CLOUD_LOCATION` env vars are set and ADC credentials are configured.
- **Concurrent session access**: `append_event()` is thread-safe. Do not call
  `get_session()` and modify state outside the event lifecycle.
- **Missing `session_id` in tool_context**: The `Runner` injects `session_id`
  automatically. If absent, the runner was not used — report a configuration error.

## Examples

### Example 1 — Full session lifecycle with DatabaseSessionService

```python
import asyncio
from google.adk.sessions import DatabaseSessionService
from google.adk.runners import Runner
from google.adk.agents import LlmAgent
from google.genai.types import Content, Part

async def main():
    session_service = DatabaseSessionService(
        db_url="sqlite+aiosqlite:///agent_sessions.db"
    )

    agent = LlmAgent(
        name="assistant",
        model="gemini-2.0-flash",
        instruction="You are a helpful assistant. User name: {user:name?}.",
    )

    runner = Runner(
        agent=agent,
        app_name="my_app",
        session_service=session_service,
    )

    # Create session with seed state
    session = await session_service.create_session(
        app_name="my_app",
        user_id="user_001",
        state={"user:name": "Alice", "onboarding_step": "welcome"},
    )
    session_id = session.id

    # Run first turn
    msg = Content(parts=[Part(text="Hello!")])
    for event in runner.run(user_id="user_001", session_id=session_id, new_message=msg):
        if event.is_final_response():
            print("Agent:", event.content)

    # Resume — retrieve updated session
    updated = await session_service.get_session(
        app_name="my_app", user_id="user_001", session_id=session_id
    )
    print("State after turn:", updated.state)

    # Cleanup
    await session_service.delete_session(
        app_name="my_app", user_id="user_001", session_id=session_id
    )

asyncio.run(main())
```

### Example 2 — Resuming a session safely

```python
async def resume_or_create(session_service, app_name, user_id, session_id):
    """Resumes an existing session or creates a new one if not found."""
    session = await session_service.get_session(
        app_name=app_name, user_id=user_id, session_id=session_id
    )
    if session is None:
        session = await session_service.create_session(
            app_name=app_name,
            user_id=user_id,
            session_id=session_id,
            state={"initialized": True},
        )
    return session
```

### Example 3 — VertexAiSessionService for GCP production

```python
from google.adk.sessions import VertexAiSessionService

REASONING_ENGINE_ID = "projects/my-project/locations/us-central1/reasoningEngines/123"

session_service = VertexAiSessionService(
    project="my-project",
    location="us-central1",
)
session = await session_service.create_session(
    app_name=REASONING_ENGINE_ID,
    user_id="user_001",
)
```

## Edge Cases

- **New session with no initial state**: `state` defaults to `{}`. Always use
  `.get(key, default)` when reading state in tools.
- **Expired long-running session**: `VertexAiSessionService` may expire old
  sessions. Implement a session refresh heartbeat for long-lived workflows.
- **`session_id` collision**: Auto-generate IDs using `uuid.uuid4()` to avoid
  collisions in high-concurrency environments.
- **Session replay after restart**: `DatabaseSessionService` preserves `events`.
  Replay is possible by reading `session.events` and reconstructing context.
- **Missing `events` in retrieved session**: Check `session.events` length before
  assuming history exists — may be a fresh session.

## Integration Notes

- **`Runner`**: Automatically manages session retrieval, event appending, and
  session state persistence for each `runner.run()` call.
- **`google-adk-conversation-state-tracking`**: Depends on this skill for
  session initialization. Always set up `SessionService` first.
- **`google-adk-checkpointing`**: Depends on `append_event()` from
  `SessionService` to save checkpoint events.
- **`google-adk-cross-session-memory`**: Uses `user:` prefixed state keys that
  are managed across sessions by the same `SessionService`.
- **ADK Schema migration**: `DatabaseSessionService` schema changed in ADK
  Python v1.22.0. Run DB migration when upgrading.
