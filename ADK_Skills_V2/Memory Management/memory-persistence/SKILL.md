---
name: memory-persistence
description: >
  Use this skill when ensuring that Google ADK agent memory survives server
  restarts, deployment updates, and session boundaries. Covers configuring
  persistent session service backends (DatabaseSessionService and
  VertexAiSessionService), implementing memory backup and recovery strategies,
  handling persistence failures gracefully, and validating that user: and
  session state persists correctly across restarts.
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

This skill implements reliable memory persistence for Google ADK agents — ensuring
that `tool_context.state`, `user:` prefixed state, and episodic memory survive
server restarts, rolling deployments, and session boundaries. In production ADK
deployments, the `InMemorySessionService` loses all memory on process exit.
This skill configures persistent backends (`DatabaseSessionService` and
`VertexAiSessionService`), validates persistence correctness, implements backup
and recovery strategies, and handles persistence failures gracefully without
disrupting agent operation.

## When to Use

- When deploying an ADK agent to production where `InMemorySessionService` loses
  data on every restart.
- When validating that `user:` state actually persists across sessions.
- When implementing memory backup for compliance or disaster recovery.
- When rolling deployments cause agents to restart mid-session.
- When the session database requires schema migration after an ADK upgrade.

## When NOT to Use

- Do not use `DatabaseSessionService` for local development — `InMemorySessionService`
  is sufficient and requires no setup.
- Do not back up every state key — only back up high-value memory (user preferences,
  booking records, episode logs).
- Do not implement persistence for `temp:` keys — they are intentionally ephemeral.

## Google ADK Context

- **`InMemorySessionService`**: Zero persistence. All state lost on process exit.
  Dev/test only.
- **`DatabaseSessionService`**: SQL-backed persistence (SQLite, PostgreSQL, MySQL).
  State committed after every agent invocation. Survives restarts.
- **`VertexAiSessionService`**: GCP-managed persistence. SLA-backed availability.
  Scalable across instances.
- **Session commit lifecycle**: After each `runner.run()` call completes (all
  events emitted), ADK commits the updated session state to the backend.
- **`user:` key persistence**: `user:` prefixed keys are persisted under the
  `user_id` namespace by the session service. Any persistent backend supports this.
- **Persistence failure handling**: If the session backend throws during commit,
  ADK raises an exception. Implement retry logic in the runner invocation wrapper.
- **Memory backup strategy**: Export critical `user:` state entries periodically
  to a secondary store (GCS bucket, database table) for disaster recovery.
- **Session schema migration**: When the state format changes across versions,
  implement a migration in `before_agent_callback` to update the state schema.

## Capabilities

- Configures persistent session backends (`Database` and `VertexAI`).
- Validates persistence by writing and reading state across sessions.
- Implements periodic backup of critical user state to a secondary store.
- Handles persistence failures with retry and circuit breaker patterns.
- Implements schema migration for state format changes.
- Validates `user:` state persistence across session boundaries.

## Step-by-Step Instructions

1. **Configure a persistent backend** (see `google-adk-memory-storage-backends`):
   ```python
   from google.adk.sessions import DatabaseSessionService
   session_service = DatabaseSessionService(db_url=os.environ["DATABASE_URL"])
   ```

2. **Validate persistence at startup**:
   ```python
   async def validate_persistence(session_service) -> bool:
       test_session = await session_service.create_session(app_name="test", user_id="health_check")
       await session_service.delete_session(app_name="test", user_id="health_check", session_id=test_session.id)
       return True
   ```

3. **Write critical state** to `user:` namespace for cross-session durability:
   ```python
   tool_context.state["user:profile_name"] = user_name
   tool_context.state["user:profile_updated_at"] = time.time()
   ```

4. **Implement a backup job** for critical user state:
   ```python
   import json
   async def backup_user_state(user_id: str, session_service, gcs_client, bucket_name: str):
       sessions = await session_service.list_sessions(app_name="my_app", user_id=user_id)
       for session in sessions.sessions[:1]:  # Use latest session for user: state
           full_session = await session_service.get_session(app_name="my_app",
                                                            user_id=user_id, session_id=session.id)
           user_state = {k: v for k, v in full_session.state.items() if k.startswith("user:")}
           blob = gcs_client.bucket(bucket_name).blob(f"backups/{user_id}/state.json")
           blob.upload_from_string(json.dumps(user_state, default=str))
   ```

5. **Implement state schema migration** in `before_agent_callback`:
   ```python
   def migrate_state_schema(callback_context: CallbackContext):
       current_version = int(callback_context.state.get("meta:state_version", 0))
       if current_version < 2:
           # Migrate v1 → v2: rename 'prefs' to 'user:preferences'
           old_prefs = callback_context.state.pop("prefs", None)
           if old_prefs:
               callback_context.state["user:preferences"] = old_prefs
           callback_context.state["meta:state_version"] = 2
       return None
   ```

6. **Add retry wrapper** around runner invocations for persistence resilience:
   ```python
   import asyncio
   async def run_with_retry(runner, user_id, session_id, msg, max_retries=3):
       for attempt in range(max_retries):
           try:
               return list(runner.run(user_id=user_id, session_id=session_id, new_message=msg))
           except Exception as exc:
               if attempt == max_retries - 1:
                   raise
               await asyncio.sleep(2 ** attempt)
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "long_term",
  "content": "user:profile_name=Alice|user:preferred_seat=aisle",
  "metadata": {"backend": "DatabaseSessionService", "backup": True},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "user:42:profile_name",
  "memory_retrieved": [
    {"key": "user:profile_name", "value": "Alice", "persisted": True},
    {"key": "user:preferred_seat", "value": "aisle", "persisted": True}
  ],
  "memory_injected": False
}
```

## Error Handling

- **Database connection failure at startup**: Fail fast with a clear error message.
  Never start without a confirmed database connection.
- **Persistence failure mid-session** (DB unavailable during invocation): Catch
  the exception in the runner wrapper. Return a graceful error to the user.
  Queue the commit for retry when the DB recovers.
- **State lost on restart** (still using InMemorySessionService): Audit backends
  by checking `type(runner.session_service).__name__`. Alert if InMemorySessionService
  is detected in production.
- **Schema migration fails** (old state keys missing): Use `state.get(old_key)`
  not `state[old_key]` — missing keys are expected during migration.
- **GCS backup upload fails**: Log the error. Do not fail the agent run — backup
  is a secondary concern.

## Examples

### Example 1 — Production-grade persistence configuration

```python
import os
import logging
from google.adk.sessions import DatabaseSessionService, VertexAiSessionService
from google.adk.runners import Runner

logger = logging.getLogger("persistence")

def create_persistent_session_service():
    """Creates a persistent session service for production deployments."""
    env = os.environ.get("DEPLOYMENT_ENV", "dev")
    if env == "gcp":
        service = VertexAiSessionService(
            project=os.environ["GOOGLE_CLOUD_PROJECT"],
            location=os.environ.get("VERTEX_AI_LOCATION", "us-central1"),
        )
        logger.info("Using VertexAiSessionService for persistence.")
    elif env in ("prod", "staging"):
        db_url = os.environ["DATABASE_URL"]
        service = DatabaseSessionService(db_url=db_url)
        logger.info(f"Using DatabaseSessionService: {db_url[:30]}...")
    else:
        from google.adk.sessions import InMemorySessionService
        service = InMemorySessionService()
        logger.warning("Using InMemorySessionService. Memory NOT persisted.")
    return service
```

### Example 2 — Startup persistence validation

```python
async def validate_persistence_health(session_service) -> dict:
    """Validates that the session backend correctly persists state."""
    TEST_KEY = "user:_health_check_value"
    TEST_VALUE = "persistence_ok"

    # Write a test value
    session = await session_service.create_session(
        app_name="health_check", user_id="sys_health",
        state={TEST_KEY: TEST_VALUE}
    )
    session_id = session.id

    # Read it back
    read_session = await session_service.get_session(
        app_name="health_check", user_id="sys_health", session_id=session_id
    )
    value_ok = read_session and read_session.state.get(TEST_KEY) == TEST_VALUE

    # Cleanup
    await session_service.delete_session(
        app_name="health_check", user_id="sys_health", session_id=session_id
    )

    return {
        "backend": type(session_service).__name__,
        "persistence_validated": value_ok,
        "status": "healthy" if value_ok else "UNHEALTHY",
    }
```

### Example 3 — State schema migration in before_agent_callback

```python
import time
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content
import logging

logger = logging.getLogger("state_migration")
CURRENT_STATE_VERSION = 3

def migrate_legacy_state(callback_context: CallbackContext) -> Optional[Content]:
    """Migrates legacy state schema to the current version."""
    version = int(callback_context.state.get("meta:state_version", 0))
    if version >= CURRENT_STATE_VERSION:
        return None  # Already up to date

    if version < 1:
        # v0 → v1: standardize preference key names
        if "user_language" in callback_context.state:
            callback_context.state["user:preferred_language"] = callback_context.state.pop("user_language")
        version = 1

    if version < 2:
        # v1 → v2: add missing default for booking count
        if "user:total_bookings" not in callback_context.state:
            callback_context.state["user:total_bookings"] = 0
        version = 2

    if version < 3:
        # v2 → v3: normalize episode log to list (was stored as JSON string in v2)
        import json
        raw = callback_context.state.get("user:episode_log", [])
        if isinstance(raw, str):
            try:
                callback_context.state["user:episode_log"] = json.loads(raw)
            except json.JSONDecodeError:
                callback_context.state["user:episode_log"] = []
        version = 3

    callback_context.state["meta:state_version"] = CURRENT_STATE_VERSION
    callback_context.state["meta:state_migrated_at"] = time.time()
    logger.info(f"State migrated to version {CURRENT_STATE_VERSION} for session {callback_context.session_id}")
    return None
```

## Edge Cases

- **Multi-replica deployment writes same session simultaneously**: Use
  database-level row locking in `DatabaseSessionService`. PostgreSQL handles
  this via SELECT FOR UPDATE on session rows.
- **Session TTL expires for long-running workflows**: Implement a session refresh
  endpoint that touches the session to reset TTL before it expires.
- **`VertexAiSessionService` quota limit exceeded**: Implement exponential backoff
  retry. For burst traffic, consider a local cache layer in front of the Vertex
  AI service.
- **State size exceeds backend limits**: Compress large state values before
  committing. Move binary/large data to GCS and store only the reference.
- **Cross-region persistence latency**: For multi-region deployments, prefer a
  regionally co-located session backend. Avoid cross-region database writes in
  the critical path.

## Integration Notes

- **`google-adk-memory-storage-backends`**: Covers the backend selection decision.
  This skill covers the operational lifecycle — backup, recovery, migration.
- **`google-adk-long-term-memory`**: `user:` state persistence relies entirely on a
  persistent backend configured by this skill.
- **`google-adk-short-term-memory`**: Session-scoped state is also persisted by
  `DatabaseSessionService` across server restarts within a session TTL.
- **`google-adk-memory-eviction-strategies`**: Eviction policies must account for
  the persistence layer — evicting a state key also removes it from the database
  on the next commit.
