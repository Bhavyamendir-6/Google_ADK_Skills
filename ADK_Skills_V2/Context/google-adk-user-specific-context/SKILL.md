---
name: google-adk-user-specific-context
description: >
  Use this skill when a Google ADK agent needs to personalize its behavior
  based on user-specific data stored in user: prefixed session state. Covers
  loading, reading, updating, and injecting user-specific context (preferences,
  profile, history) into agent instructions and tool calls using tool_context.user_id
  and user: scoped state keys.
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

This skill enables ADK agents to read and write user-specific context — such
as display name, language preference, role, subscription tier, and behavioral
history — using the `user:` prefix in `session.state`. It ensures that all
personalization data is correctly scoped to the individual user, persisted
across sessions by compatible `SessionService` backends, and injected into
agent instructions and tool calls in a type-safe manner.

## When to Use

- When the agent's behavior, tone, or capabilities must vary per user.
- When user preferences, profile data, or role information must be accessible
  in tool functions via `tool_context.state`.
- When writing user-specific data discovered mid-conversation back to
  persistent state for future sessions.
- When implementing multi-tenant agents where each user has distinct context.

## When NOT to Use

- Do not use `user:` prefix for session-specific temporary data — use no prefix.
- Do not store raw PII (passwords, payment card numbers) in `user:` state
  without encryption.
- Do not use `InMemorySessionService` if `user:` state must persist across
  server restarts.

## Google ADK Context

- **`user:` prefix**: State keys prefixed with `user:` are scoped to `user_id`
  and are shared across all sessions for that user (within the same `app_name`).
  Persisted by `DatabaseSessionService` and `VertexAiSessionService`.
- **`tool_context.user_id`**: Read-only string. The current user's unique ID.
  Available in all tool functions. Use it to key external user data lookups.
- **`tool_context.state`**: Dict-like access to the merged session + user state.
  Write `tool_context.state["user:key"] = value` to persist user state changes.
- **`session.state` on retrieval**: The `SessionService` merges `user:` scoped
  state into `session.state` on `get_session()`, making all user state visible.
- **ADK Execution Lifecycle**: `user:` state is loaded at session start and
  flushed via `append_event()` after tool/callback updates.

## Capabilities

- Read user profile data from `tool_context.state["user:key"]` in tool functions.
- Write user preference updates to `tool_context.state["user:key"]` with persistence.
- Inject user-specific context into LlmAgent instructions via `{user:key?}`.
- Load user context from external systems (CRM, DB) and store in `user:` state.
- Validate user role/permission from `user:` state in tool functions.
- Automatically detect first-time users (no `user:` state keys present).

## Step-by-Step Instructions

1. **Initialize user state** at session creation with known profile data:
   ```python
   session = await session_service.create_session(
       app_name="my_app",
       user_id="user_123",
       state={
           "user:name": "Alice",
           "user:role": "premium",
           "user:preferred_language": "en",
           "user:timezone": "America/New_York",
       },
   )
   ```

2. **Detect first-time users** in a tool or callback:
   ```python
   is_new_user = "user:name" not in tool_context.state
   ```

3. **Read user context in tool functions**:
   ```python
   user_name = tool_context.state.get("user:name", "User")
   user_role = tool_context.state.get("user:role", "standard")
   user_lang = tool_context.state.get("user:preferred_language", "en")
   ```

4. **Write updated user state in tool functions**:
   ```python
   tool_context.state["user:preferred_language"] = new_language
   tool_context.state["user:last_active_at"] = time.time()
   ```

5. **Load external user profile** via a dedicated tool and cache in `user:` state:
   ```python
   profile = crm_client.get_user(tool_context.user_id)
   tool_context.state["user:company"] = profile.company
   tool_context.state["user:subscription"] = profile.subscription_tier
   ```

6. **Inject user context into agent instruction**:
   ```python
   LlmAgent(instruction=(
       "You are assisting {user:name?} (role: {user:role?}). "
       "Respond in {user:preferred_language?}."
   ))
   ```

7. **Guard tool actions by user role**:
   ```python
   if tool_context.state.get("user:role") != "admin":
       return {"error": "Permission denied. Admin role required.", "error_type": "permanent"}
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_xyz",
  "state": {
    "user:name": "Alice",
    "user:role": "premium",
    "user:preferred_language": "en",
    "user:timezone": "America/New_York",
    "user:session_count": 12,
    "user:last_active_at": 1700000000.0
  },
  "metadata": {}
}
```

## Output Format

```python
{
  "state_updated": True,
  "state_key": "user:preferred_language",
  "state_value": "fr",
  "scope": "user",
  "user_id": "user_xyz"
}
```

## Error Handling

- **`user:` key missing for new user**: Default gracefully. Log a warning and
  trigger a user profile bootstrap tool if a critical key (e.g., `user:name`)
  is absent.
- **External profile load fails**: Return a transient error and proceed with
  defaults. Do not block agent execution on CRM failures.
- **Stale `user:` state** (preference changed in another channel): Add a
  `user:profile_version` key. Compare to external profile version on each session
  start and refresh if stale.
- **Role-based access violation**: Return `{"error": "Permission denied."}` as
  a tool result. Do NOT raise an exception.
- **`user_id` is None or empty**: Reject the session creation request. A valid
  `user_id` is required for all user-scoped operations.

## Examples

### Example 1 — User profile initialization and access

```python
from google.adk.tools import ToolContext
import time

def load_user_profile(tool_context: ToolContext) -> dict:
    """Loads the user profile from an external CRM and caches it in user: state.

    Args:
        tool_context (ToolContext): ADK tool context with user_id and state.

    Returns:
        dict: Contains 'name', 'role', and 'language'.
    """
    user_id = tool_context.user_id

    # Check if profile already cached
    if "user:profile_loaded" in tool_context.state:
        return {
            "name": tool_context.state.get("user:name", "User"),
            "role": tool_context.state.get("user:role", "standard"),
            "language": tool_context.state.get("user:preferred_language", "en"),
        }

    # Simulate external CRM lookup
    profile = {"name": "Alice", "role": "premium", "language": "en", "company": "Acme"}

    tool_context.state["user:name"] = profile["name"]
    tool_context.state["user:role"] = profile["role"]
    tool_context.state["user:preferred_language"] = profile["language"]
    tool_context.state["user:company"] = profile["company"]
    tool_context.state["user:profile_loaded"] = True
    tool_context.state["user:profile_loaded_at"] = time.time()

    return {"name": profile["name"], "role": profile["role"], "language": profile["language"]}
```

### Example 2 — Role-gated tool

```python
def get_admin_report(report_type: str, tool_context: ToolContext) -> dict:
    """Retrieves an admin-only report. Requires admin role.

    Args:
        report_type (str): Type of report: 'usage', 'billing', or 'audit'.
        tool_context (ToolContext): ADK tool context.

    Returns:
        dict: Report data or permission error.
    """
    role = tool_context.state.get("user:role", "standard")
    if role != "admin":
        return {
            "error": f"Permission denied. Admin role required, current role: '{role}'.",
            "error_type": "permanent",
            "recoverable": False,
        }
    return {"report_type": report_type, "data": f"[{report_type} report data]"}
```

### Example 3 — Personalized LlmAgent

```python
from google.adk.agents import LlmAgent

personalized_agent = LlmAgent(
    name="personalized_assistant",
    model="gemini-2.0-flash",
    instruction=(
        "You are helping {user:name?} from {user:company?}.\n"
        "Their subscription tier is: {user:role?}.\n"
        "Always respond in {user:preferred_language?}.\n"
        "Be concise and professional."
    ),
    tools=[load_user_profile, get_admin_report],
)
```

## Edge Cases

- **Multiple users sharing a session**: ADK sessions are 1:1 with a `user_id`.
  Do not attempt to share a session between multiple users.
- **User switches language mid-session**: Update `user:preferred_language` via
  tool call. The changed value will be injected in the next turn's instruction.
- **User upgrades subscription mid-conversation**: Trigger a profile refresh
  tool to update `user:role`. The agent will use the new role in subsequent turns.
- **Anonymous users (no user_id)**: Use a temporary session-scoped user ID
  (e.g., UUID). Migrate to a real user ID on authentication and copy `user:` state.

## Integration Notes

- **`google-adk-context-injection`**: Injects `user:` scoped state values into
  `LlmAgent.instruction` via `{user:key?}` placeholders.
- **`google-adk-cross-session-memory`**: Provides the `user:` key persistence
  mechanism. Apply the cross-session memory skill for session bootstrapping.
- **`google-adk-metadata-handling`**: Use metadata to store non-state user
  attributes (request-scoped user agent, IP, etc.).
- **External CRM/DB**: Always cache external profile data in `user:` state
  after first load to avoid repeated external calls.
