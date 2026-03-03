---
name: memory-eviction-strategies
description: >
  Use this skill when implementing policies for removing outdated, low-relevance,
  or excess memory entries from a Google ADK agent's state storage to prevent
  unbounded growth. Covers FIFO, LRU, time-based TTL, relevance-score-based,
  and size-based eviction strategies applied to tool_context.state episode logs
  and user: memory entries.
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

This skill implements memory eviction policies for Google ADK agents to prevent
unbounded growth of session state, episode logs, and long-term memory stores.
Without eviction, memory grows monotonically — degrading performance, exceeding
storage limits, and bloating context window usage. This skill covers five eviction
strategies: FIFO (first-in-first-out), LRU (least-recently-used), TTL (time-to-live),
relevance-score-based, and size-triggered eviction. Each strategy is implemented
as a reusable function applied to `tool_context.state` memory entries.

## When to Use

- When the episode log approaches MAX_EPISODES (e.g., > 80 of 100 limit).
- When `user:` state memory entries grow too large for efficient context injection.
- When session state values include large tool result caches that become stale.
- When implementing a vector store with a per-user entry limit.
- Proactively at the end of each session to archive and evict old memories.

## When NOT to Use

- Do not evict memory entries that are referenced as bookings, payment records,
  or other compliance-relevant artifacts — archive instead of delete.
- Do not apply eviction based on content semantics without human review for
  high-value memories (e.g., premium subscription data).
- Do not evict session state mid-turn — apply eviction only in `before_agent_callback`
  or after `is_final_response()`.

## Google ADK Context

- **`tool_context.state["user:episode_log"]`**: The primary list requiring eviction
  management. Cap at MAX_EPISODES with FIFO eviction.
- **`tool_context.state` key expiry**: ADK does not natively support per-key TTL.
  Implement TTL manually by storing `{key}_expires_at` timestamps alongside values.
- **`before_agent_callback`**: The preferred hook for running eviction checks before
  each invocation.
- **`tool_context.user_id`**: Used to scope eviction to a specific user's memory.
- **TTL eviction pattern**: On read, check `state.get("{key}_expires_at")`; if
  expired, delete the entry and return None.
- **Relevance-based eviction**: Score each memory entry by access frequency,
  recency, and type importance. Evict lowest-scoring entries first.
- **Vector store eviction**: Remove embedding entries that fall below a relevance
  threshold or exceed the user's corpus size limit.

## Capabilities

- Implements FIFO eviction for episode logs (oldest entries first).
- Implements TTL-based expiry for cache entries in session state.
- Implements LRU eviction using access frequency tracking.
- Implements size-triggered eviction when state value exceeds a size threshold.
- Implements relevance-score-based eviction for episodic and semantic memory.
- Archives evicted entries to `user:evicted_log` before deletion.

## Step-by-Step Instructions

1. **Choose the eviction strategy** for each memory type:
   - Episode log → FIFO with MAX_EPISODES cap
   - Cache entries → TTL eviction
   - User preferences (LRU) → size-bounded with LRU
   - Large tool results → size-triggered

2. **Implement FIFO eviction** for episode logs:
   ```python
   MAX_EPISODES = 100
   log = tool_context.state.get("user:episode_log", [])
   if len(log) > MAX_EPISODES:
       log = log[-MAX_EPISODES:]  # Keep last MAX_EPISODES entries
       tool_context.state["user:episode_log"] = log
   ```

3. **Implement TTL eviction** for cache entries:
   ```python
   import time
   def get_with_ttl(key: str, tool_context, default=None):
       expires_key = f"{key}_expires_at"
       if tool_context.state.get(expires_key, 0) < time.time():
           tool_context.state.pop(key, None)
           tool_context.state.pop(expires_key, None)
           return default
       return tool_context.state.get(key, default)
   ```

4. **Implement size-triggered eviction** for large values:
   ```python
   MAX_VALUE_CHARS = 5000
   if len(str(tool_context.state.get(key, ""))) > MAX_VALUE_CHARS:
       del tool_context.state[key]
   ```

5. **Archive before eviction** for compliance:
   ```python
   evicted = log[:len(log) - MAX_EPISODES]
   tool_context.state["user:evicted_episodes_latest"] = evicted[-10:]  # Last 10 evicted
   ```

6. **Apply eviction in `before_agent_callback`** to run it proactively each turn.

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "episodic",
  "content": "eviction check at turn 35",
  "metadata": {"strategy": "fifo", "max_entries": 100, "current_entries": 105},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": False,
  "memory_id": None,
  "memory_retrieved": [{"evicted_count": 5, "remaining_count": 100, "strategy": "fifo"}],
  "memory_injected": False
}
```

## Error Handling

- **Eviction removes critically needed entry**: Implement a `pinned` flag in
  the episode metadata. Pinned entries are never evicted.
- **TTL-expired entry still accessed**: Always check TTL on read, not just on
  write. Apply TTL enforcement consistently across all read paths.
- **FIFO eviction removes a compliance-relevant record**: Archive evicted entries
  to an external database before deletion. Never hard-delete compliance records.
- **Eviction triggered mid-turn** (tool updates state that triggers eviction):
  Apply eviction only in `before_agent_callback` or dedicated eviction tool calls.

## Examples

### Example 1 — Comprehensive eviction manager

```python
import time
import json
import logging
from typing import Optional
from google.adk.tools import ToolContext

logger = logging.getLogger("memory_eviction")

MAX_EPISODE_LOG = 100
MAX_CACHE_TTL_SECONDS = 3600  # 1 hour cache TTL
MAX_STATE_VALUE_CHARS = 8000

def run_eviction_cycle(tool_context: ToolContext) -> dict:
    """Runs all configured eviction strategies on the current session state.

    Args:
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'evicted_episodes' (int), 'expired_caches' (int).
    """
    evicted_episodes = 0
    expired_caches = 0

    # 1. FIFO eviction on episode log
    raw = tool_context.state.get("user:episode_log", [])
    log = json.loads(raw) if isinstance(raw, str) else (raw if isinstance(raw, list) else [])
    if len(log) > MAX_EPISODE_LOG:
        n_evict = len(log) - MAX_EPISODE_LOG
        evicted = log[:n_evict]
        log = log[n_evict:]
        tool_context.state["user:episode_log"] = log
        # Archive last 5 evicted to user state for reference
        tool_context.state["user:last_evicted_episodes"] = evicted[-5:]
        evicted_episodes = n_evict
        logger.info(f"FIFO eviction: {n_evict} episodes evicted. user_id={tool_context.user_id}")

    # 2. TTL eviction on cache keys
    now = time.time()
    cache_keys = [k for k in list(tool_context.state.keys()) if k.startswith("cache:")]
    for key in cache_keys:
        expires_key = f"{key}_expires_at"
        expiry = float(tool_context.state.get(expires_key, 0))
        if expiry and expiry < now:
            tool_context.state.pop(key, None)
            tool_context.state.pop(expires_key, None)
            expired_caches += 1
            logger.debug(f"TTL expired: {key}")

    # 3. Size-triggered eviction on oversized state values
    for key in list(tool_context.state.keys()):
        value = tool_context.state.get(key, "")
        if isinstance(value, str) and len(value) > MAX_STATE_VALUE_CHARS:
            logger.warning(f"Size-triggered eviction: {key} ({len(value)} chars). user_id={tool_context.user_id}")
            del tool_context.state[key]

    return {"evicted_episodes": evicted_episodes, "expired_caches": expired_caches}
```

### Example 2 — Pinned episode protection

```python
def append_episode_with_pin(episode: dict, pinned: bool, tool_context: ToolContext, max_episodes: int = 100):
    """Appends an episode to the log, protecting pinned entries from FIFO eviction."""
    episode["pinned"] = pinned
    raw = tool_context.state.get("user:episode_log", [])
    log = json.loads(raw) if isinstance(raw, str) else (raw if isinstance(raw, list) else [])
    log.append(episode)
    if len(log) > max_episodes:
        # Evict oldest non-pinned entries first
        non_pinned = [e for e in log if not e.get("pinned", False)]
        pinned_entries = [e for e in log if e.get("pinned", False)]
        n_evict = len(log) - max_episodes
        if len(non_pinned) > n_evict:
            non_pinned = non_pinned[n_evict:]  # FIFO from non-pinned
        log = pinned_entries + non_pinned
    tool_context.state["user:episode_log"] = log
```

## Edge Cases

- **All entries are pinned** (nothing to evict): Log a warning. Storage will
  exceed the limit. Notify operations team to review pinning policy.
- **Eviction loop** (eviction triggers another eviction): Apply eviction once
  per `before_agent_callback` invocation — guard with a flag:
  `if state.get("temp:eviction_ran"): return`.
- **Concurrent sessions for same user** apply eviction simultaneously: Race
  condition on the episode log. Use database-level transactions in
  `DatabaseSessionService` for atomic eviction.

## Integration Notes

- **`google-adk-episodic-memory`**: Episode logs are the primary data structure
  requiring FIFO/TTL eviction.
- **`google-adk-memory-compression`**: Apply compression before eviction —
  compression may eliminate the need to evict by reducing entry sizes.
- **`google-adk-vector-memory`**: Vector store eviction removes embeddings from
  the collection. Use the same strategies (TTL, size limit, relevance score).
- **`google-adk-memory-persistence`**: Eviction only affects the live state.
  Archived entries must be committed to the persistent backend before deletion.
