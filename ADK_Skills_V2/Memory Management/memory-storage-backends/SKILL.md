---
name: memory-storage-backends
description: >
  Use this skill when selecting and configuring the storage backend for Google
  ADK agent session state and memory. Covers the three built-in ADK session
  service backends (InMemorySessionService, DatabaseSessionService,
  VertexAiSessionService), their tradeoffs, configuration, and when to use
  each in development vs. production deployments.
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

This skill selects and configures the correct storage backend for Google ADK
agent session state and memory persistence. The backend determines where
`tool_context.state`, `user:` prefixed state, and `app:` state are persisted —
in-process memory, a relational database, or Google Cloud's Vertex AI session
storage. The wrong backend choice leads to memory loss on restart (using
`InMemorySessionService` in production) or unnecessary complexity (using
`DatabaseSessionService` in development). This skill provides a decision
framework and configuration guide for each backend.

## When to Use

- When initializing a new ADK application and selecting a session service.
- When deploying to production and `InMemorySessionService` must be replaced.
- When configuring multi-instance deployment where session state must be shared
  across replicas.
- When migrating from development to production session storage.
- When Vertex AI-native session persistence is required for GCP deployment.

## When NOT to Use

- Do not use `DatabaseSessionService` for local development/testing — use
  `InMemorySessionService` for simplicity.
- Do not use `InMemorySessionService` in any multi-instance or production deployment.
- Do not store sensitive secrets (API keys, passwords) in any session backend —
  use environment variables or a secrets manager.

## Google ADK Context

- **`InMemorySessionService`**: In-process dict-based storage. Zero configuration.
  All state lost on process restart. Use only for development and testing.
- **`DatabaseSessionService`**: Relational database backend (SQLite, PostgreSQL,
  MySQL). Requires a database URL. State persists across restarts and replica instances.
- **`VertexAiSessionService`**: Google Cloud Vertex AI-managed session storage.
  GCP-native, scalable, no database to manage. Requires `GOOGLE_CLOUD_PROJECT` env var.
- **`tool_context.state` persistence**: The session service backend is the layer
  that persists `tool_context.state` reads and writes across turns and sessions.
- **ADK Agent Execution Lifecycle**:
  1. `Runner.__init__` receives a `session_service`.
  2. On each `runner.run()` call, state is loaded from the backend.
  3. After each turn, state changes are committed to the backend.

## Capabilities

- Configures `InMemorySessionService` for development/testing.
- Configures `DatabaseSessionService` for production with persistent SQL storage.
- Configures `VertexAiSessionService` for GCP-native deployments.
- Provides migration guidance from in-memory to persistent backends.
- Enables multi-instance state sharing via database-backed session service.
- Validates backend connectivity at startup.

## Step-by-Step Instructions

1. **Choose the backend** based on environment:
   - `dev/test`: `InMemorySessionService`
   - `prod (self-hosted)`: `DatabaseSessionService` with PostgreSQL
   - `prod (GCP)`: `VertexAiSessionService`

2. **Configure `InMemorySessionService`** (no dependencies):
   ```python
   from google.adk.sessions import InMemorySessionService
   session_service = InMemorySessionService()
   ```

3. **Configure `DatabaseSessionService`** (SQLAlchemy-compatible DB URL):
   ```python
   from google.adk.sessions import DatabaseSessionService
   import os
   DB_URL = os.environ["DATABASE_URL"]  # e.g., "postgresql+asyncpg://user:pass@host/db"
   session_service = DatabaseSessionService(db_url=DB_URL)
   ```

4. **Configure `VertexAiSessionService`** (GCP-native):
   ```python
   from google.adk.sessions import VertexAiSessionService
   import os
   PROJECT_ID = os.environ["GOOGLE_CLOUD_PROJECT"]
   LOCATION = os.environ.get("GOOGLE_CLOUD_LOCATION", "us-central1")
   session_service = VertexAiSessionService(project=PROJECT_ID, location=LOCATION)
   ```

5. **Pass the session service to the Runner**:
   ```python
   from google.adk.runners import Runner
   runner = Runner(agent=agent, app_name="my_app", session_service=session_service)
   ```

6. **Validate connectivity at startup**:
   ```python
   async def validate_backend(session_service):
       test_session = await session_service.create_session(app_name="test", user_id="health_check")
       await session_service.delete_session(app_name="test", user_id="health_check", session_id=test_session.id)
       return True
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "short_term",
  "content": "backend=postgresql",
  "metadata": {"backend_type": "database", "db_url": "postgresql://..."},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "sess_abc123:state",
  "memory_retrieved": [{"backend": "DatabaseSessionService", "connected": True}],
  "memory_injected": False
}
```

## Error Handling

- **Database connection failure**: Raise at startup with a clear message: "Failed
  to connect to session database. Check DATABASE_URL environment variable."
- **`InMemorySessionService` in production** (state lost on restart): Audit all
  deployments for session service type. Fail CI/CD if `InMemorySessionService`
  is used in non-dev environments.
- **`VertexAiSessionService` auth failure** (`403 Forbidden`): Verify that the
  service account has `roles/aiplatform.user` or `vertexai.sessions.*` permissions.
- **Database schema migration failure**: ADK's `DatabaseSessionService` creates
  schema on first run. Ensure the database user has CREATE TABLE permissions.
- **Concurrent session writes** (multiple replicas): The database backend handles
  concurrency via transactions. `InMemorySessionService` has no concurrency guarantees.

## Examples

### Example 1 — Environment-based backend selection

```python
import os
from google.adk.sessions import InMemorySessionService, DatabaseSessionService, VertexAiSessionService

def create_session_service():
    """Creates the appropriate session service backend based on deployment environment."""
    env = os.environ.get("DEPLOYMENT_ENV", "dev")
    if env == "dev":
        return InMemorySessionService()
    elif env == "gcp":
        project = os.environ["GOOGLE_CLOUD_PROJECT"]
        location = os.environ.get("GOOGLE_CLOUD_LOCATION", "us-central1")
        return VertexAiSessionService(project=project, location=location)
    else:  # prod (self-hosted)
        db_url = os.environ["DATABASE_URL"]
        return DatabaseSessionService(db_url=db_url)

session_service = create_session_service()
```

### Example 2 — Production PostgreSQL configuration

```python
import os
from google.adk.sessions import DatabaseSessionService
from google.adk.runners import Runner
from google.adk.agents import LlmAgent

# Production: PostgreSQL via asyncpg
DB_URL = os.environ.get(
    "DATABASE_URL",
    "postgresql+asyncpg://adk_user:secure_pass@db.example.com:5432/adk_sessions"
)

session_service = DatabaseSessionService(db_url=DB_URL)

agent = LlmAgent(name="production_agent", model="gemini-2.0-flash", instruction="...")
runner = Runner(agent=agent, app_name="production_app", session_service=session_service)
```

### Example 3 — Vertex AI session service (GCP-native)

```python
import os
from google.adk.sessions import VertexAiSessionService
from google.adk.runners import Runner

session_service = VertexAiSessionService(
    project=os.environ["GOOGLE_CLOUD_PROJECT"],
    location=os.environ.get("VERTEX_AI_LOCATION", "us-central1"),
)

runner = Runner(agent=agent, app_name="gcp_app", session_service=session_service)
```

## Edge Cases

- **SQLite in production** (DatabaseSessionService with SQLite): SQLite does not
  support multiple concurrent writers — use PostgreSQL in production.
- **Session TTL and eviction**: `VertexAiSessionService` applies Google Cloud's
  session TTL. Implement session refresh for long-lived user sessions.
- **State schema migration** (session state format changes across versions):
  Store a `meta:state_version` key in state and migrate on read if needed.
- **Backup and recovery**: For `DatabaseSessionService`, set up regular database
  backups. For `VertexAiSessionService`, rely on GCP's managed backup policies.

## Integration Notes

- **`google-adk-memory-persistence`**: This skill covers the backend selection;
  `memory-persistence` covers the lifecycle, backup, and recovery patterns.
- **`google-adk-short-term-memory`**: All session-scoped state is persisted via
  the session service configured by this skill.
- **`google-adk-long-term-memory`**: `user:` prefixed state requires a persistent
  backend (`DatabaseSessionService` or `VertexAiSessionService`).
- **`google-adk-context-window-management`**: Large state values in any backend
  still consume context window space when injected. Keep injected values concise.
