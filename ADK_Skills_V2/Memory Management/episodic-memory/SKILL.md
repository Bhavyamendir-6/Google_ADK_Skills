---
name: episodic-memory
description: >
  Use this skill when a Google ADK agent needs to store and retrieve discrete
  records of past interactions, events, or completed tasks — organized as
  time-stamped episodes. Covers recording episodic memories in session.state
  as structured event logs, implementing episode indexing by timestamp and
  type, and retrieving relevant episodes for contextual reasoning.
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

This skill implements episodic memory for Google ADK agents — the ability to
remember and reason about discrete past events, interactions, and completed
tasks. Unlike short-term session state (transient within one session) or semantic
memory (general knowledge), episodic memory is a structured record of "what happened
when" — time-stamped events such as completed bookings, past search queries,
user feedback instances, and error events. Episodic memories are stored as
structured records in `user:` state or a persistent database and retrieved by
recency, type, or relevance.

## When to Use

- When the agent must reference specific past events ("When did I last book a
  flight?", "What was my most recent search?").
- When building an event log of agent actions for auditability.
- When the agent personalizes responses based on the user's interaction history
  (e.g., avoid recommending a hotel they disliked).
- When implementing agent learning from feedback — recording which recommendations
  were accepted and which were rejected.
- When the agent must detect repeated user intents across sessions.

## When NOT to Use

- Do not use episodic memory for general factual knowledge about the world — use
  semantic memory for that.
- Do not store entire conversation transcripts as episodes — summarize and
  extract key events only.
- Do not grow the episode log unboundedly — implement eviction after N episodes
  or compress old episodes.

## Google ADK Context

- **`tool_context.state["user:episode_log"]`**: Primary store for episodic
  memories. A JSON-serialized list of episode dicts with `type`, `content`,
  `timestamp`, and `metadata`.
- **`tool_context.user_id`**: Used to scope episode logs to a specific user.
- **`tool_context.session_id`**: Used to group episodes from the same session.
- **Episode schema**: `{"type": str, "content": str, "timestamp": float, "session_id": str, "metadata": dict}`
- **ADK Agent Execution Lifecycle**: Episodes are written at tool call boundaries
  (when a meaningful event occurs) and read at session start or on demand.
- **`InstructionProvider`**: Used to inject relevant recent episodes into the
  agent instruction for context.
- **Episode indexing**: By timestamp (most recent first), by type filter, or
  by semantic similarity (vector-indexed episodic memory).

## Capabilities

- Records discrete past events as time-stamped episode dicts.
- Stores episode logs persistently in `user:` state across sessions.
- Retrieves episodes by type, recency, session, or keyword.
- Injects recent or relevant episodes into agent context.
- Implements episode count limits and eviction strategies.
- Builds user interaction timelines from episodes.

## Step-by-Step Instructions

1. **Define the episode schema**:
   ```python
   from typing import TypedDict
   class Episode(TypedDict):
       type: str
       content: str
       timestamp: float
       session_id: str
       metadata: dict
   ```

2. **Write an episode** when a significant event occurs:
   ```python
   import time, json
   episode = {"type": "booking_completed", "content": "JFK→LHR BA178 2024-03-15",
              "timestamp": time.time(), "session_id": tool_context.session_id, "metadata": {}}
   log = tool_context.state.get("user:episode_log", [])
   if isinstance(log, str): log = json.loads(log)  # deserialize if stored as string
   log.append(episode)
   if len(log) > 100: log = log[-100:]  # keep last 100 episodes
   tool_context.state["user:episode_log"] = log
   ```

3. **Retrieve recent episodes**:
   ```python
   log = tool_context.state.get("user:episode_log", [])
   recent = sorted(log, key=lambda e: e["timestamp"], reverse=True)[:10]
   ```

4. **Filter by type**:
   ```python
   bookings = [e for e in log if e["type"] == "booking_completed"]
   ```

5. **Inject episodes into instruction** via `InstructionProvider`:
   ```python
   def instruction_with_episodes(ctx: ReadonlyContext) -> str:
       log = ctx.state.get("user:episode_log", [])[-5:]
       history = "\n".join(f"- [{e['type']}]: {e['content']}" for e in reversed(log))
       return f"Recent history:\n{history}\n\nHelp the user with their current request."
   ```

6. **Evict old episodes** (> 30 days) to prevent unbounded growth.

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "episodic",
  "content": "Completed booking JFK→LHR BA178 for 2024-03-15",
  "metadata": {"booking_ref": "BOOK-92834", "price_usd": 520.0},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "user:42:episode_log:idx_25",
  "memory_retrieved": [
    {"type": "booking_completed", "content": "JFK→LHR BA178", "timestamp": 1709900000}
  ],
  "memory_injected": True
}
```

## Error Handling

- **Episode log too large**: Enforce a max episode count (100) with FIFO eviction.
  Store summary of older episodes in a `user:episode_archive` key.
- **Corrupted episode log** (JSON parse failure): Wrap read in try/except.
  If parsing fails, start a fresh log and log a warning.
- **Missing session_id in episode**: Always pass `tool_context.session_id` when
  creating episodes — essential for grouping and filtering.
- **Episode type typo**: Define an enum or constant set of valid episode types
  and validate on write.
- **Missing memory backend**: Without `DatabaseSessionService`, `user:` episodes
  don't survive process restarts. Always confirm the backend is persistent.

## Examples

### Example 1 — Complete episode record and retrieval system

```python
import json
import time
import logging
from typing import Optional
from google.adk.tools import ToolContext

logger = logging.getLogger("episodic_memory")
MAX_EPISODES = 100
EPISODE_LOG_KEY = "user:episode_log"

def record_episode(episode_type: str, content: str, tool_context: ToolContext,
                   metadata: Optional[dict] = None) -> dict:
    """Records a time-stamped episode to the user's episodic memory log.

    Args:
        episode_type (str): Event type (e.g., booking_completed, search_performed).
        content (str): Human-readable description of the event.
        tool_context (ToolContext): ADK tool context.
        metadata (dict, optional): Additional structured data for the episode.
    Returns:
        dict: Contains 'episode_index' (int), 'total_episodes' (int).
    """
    episode = {
        "type": episode_type,
        "content": content,
        "timestamp": time.time(),
        "session_id": tool_context.session_id,
        "metadata": metadata or {},
    }
    raw = tool_context.state.get(EPISODE_LOG_KEY, [])
    try:
        log = json.loads(raw) if isinstance(raw, str) else (raw if isinstance(raw, list) else [])
    except (json.JSONDecodeError, TypeError):
        logger.warning("Corrupted episode log. Starting fresh.")
        log = []

    log.append(episode)
    if len(log) > MAX_EPISODES:
        log = log[-MAX_EPISODES:]  # FIFO eviction

    tool_context.state[EPISODE_LOG_KEY] = log
    return {"episode_index": len(log) - 1, "total_episodes": len(log)}


def get_recent_episodes(episode_type: Optional[str] = None, limit: int = 10,
                         tool_context: ToolContext = None) -> dict:
    """Retrieves recent episodes, optionally filtered by type.

    Args:
        episode_type (str, optional): Filter by episode type.
        limit (int): Maximum number of episodes to return.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'episodes' (list), 'total' (int).
    """
    raw = tool_context.state.get(EPISODE_LOG_KEY, [])
    log = json.loads(raw) if isinstance(raw, str) else (raw if isinstance(raw, list) else [])
    if episode_type:
        log = [e for e in log if e.get("type") == episode_type]
    recent = sorted(log, key=lambda e: e.get("timestamp", 0), reverse=True)[:limit]
    return {"episodes": recent, "total": len(log)}
```

### Example 2 — Injecting episodic context into instruction

```python
import json
from google.adk.agents.readonly_context import ReadonlyContext

def episode_aware_instruction(ctx: ReadonlyContext) -> str:
    """Builds instruction enriched with recent episodic memory."""
    raw = ctx.state.get("user:episode_log", [])
    log = json.loads(raw) if isinstance(raw, str) else (raw if isinstance(raw, list) else [])
    recent = sorted(log, key=lambda e: e.get("timestamp", 0), reverse=True)[:5]

    if recent:
        history_lines = [f"- [{e['type']}] {e['content']}" for e in reversed(recent)]
        history_section = "RECENT HISTORY:\n" + "\n".join(history_lines)
    else:
        history_section = "RECENT HISTORY: No prior interactions recorded."

    return f"""
You are FlightBot for AcmeAir.
{history_section}

Use this history to personalize your responses.
For example, if the user recently booked a route, proactively offer return flights.
"""
```

## Edge Cases

- **User requests deletion** of their episode history: Implement a `clear_episode_log`
  tool that sets `user:episode_log` to `[]` for GDPR compliance.
- **Duplicate episodes** (same event recorded twice in one turn): Add a dedup
  check using `(type, content, session_id)` hash before appending.
- **Very long episode content** (> 500 chars): Truncate to 500 chars on write.
  Full data can be stored in metadata or an external store keyed by episode ID.
- **Multiple episode types** with different schema fields: Use a `type`-discriminated
  schema with a common base and type-specific `metadata`.

## Integration Notes

- **`google-adk-long-term-memory`**: Episode logs use `user:` state — same
  persistence backend. `long-term-memory` skill covers the persistence setup.
- **`google-adk-memory-eviction-strategies`**: Eviction of older episodes when
  the log exceeds MAX_EPISODES is governed by that skill.
- **`google-adk-vector-memory`**: Index episode content as embeddings for
  semantic similarity retrieval — "find bookings similar to my recent one".
- **`google-adk-memory-summarization`**: Summarize all episodes from a session
  into a compact `user:session_summary_N` on session end.
