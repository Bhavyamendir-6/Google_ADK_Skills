---
name: memory-indexing
description: >
  Use this skill when building indexing structures over Google ADK agent memory
  for efficient non-vector retrieval. Covers creating inverted indexes, type-based
  indexes, and timestamp-based indexes over tool_context.state memory entries;
  maintaining index consistency; and querying memory by index for fast, targeted
  retrieval without semantic embedding overhead.
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

This skill implements memory indexing for Google ADK agents — building lightweight
index structures over stored memories to enable fast, targeted retrieval without
full-scan or vector embedding overhead. While vector similarity retrieval handles
semantic queries, many retrieval needs are structural: "find all episodes of type
booking_completed," "find memories from the last 7 days," "find all memories
mentioning airline BA." Memory indexing maintains these secondary lookup structures
in `tool_context.state` alongside the primary memory store.

## When to Use

- When the agent retrieves memories by known categorical criteria (type, tag,
  date range) and vector search is too slow or expensive.
- When building a multi-field filter on the episode log (e.g., type + date range).
- When memory retrieval speed is a latency bottleneck and an index can short-circuit
  a full scan.
- When building an inverted index for keyword-based memory lookup.
- When maintaining a type-to-episode-id index alongside the episode log.

## When NOT to Use

- Do not build indexes for memory collections with < 100 entries — direct linear
  scan is faster and simpler.
- Do not maintain complex B-tree style indexes in `tool_context.state` — it is
  not a database. For complex indexing needs, use an external database.
- Do not apply indexing to frequently mutated data where index maintenance overhead
  exceeds retrieval benefit.

## Google ADK Context

- **`tool_context.state["user:type_index"]`**: A dict mapping `episode_type` →
  `[episode_index, ...]` for O(1) type-based retrieval.
- **`tool_context.state["user:date_index"]`**: Maps date string (`YYYY-MM-DD`) →
  `[episode_index, ...]` for date-based filtering.
- **`tool_context.state["user:keyword_index"]`**: An inverted index mapping
  keyword → [episode_index, ...].
- **Index consistency**: Update all indexes atomically when adding or removing
  memory entries.
- **Index eviction**: When episode entries are evicted, their index entries must
  also be removed.
- **`app:` prefix for shared indexes**: Application-level semantic memory (e.g.,
  FAQ knowledge base) can be indexed in `app:` state for shared retrieval.

## Capabilities

- Creates and maintains type-based inverted indexes over episode logs.
- Creates timestamp-based date indexes for temporal filtering.
- Creates keyword inverted indexes for text search.
- Queries indexes for O(1) type retrieval and O(N_keyword) keyword retrieval.
- Maintains index consistency on memory write and eviction.
- Combines index retrieval with state-based episode lookup.

## Step-by-Step Instructions

1. **Define the index schema**:
   ```python
   # Type index: {episode_type: [episode_idx, ...]}
   # Date index: {"YYYY-MM-DD": [episode_idx, ...]}
   # Keyword index: {"keyword": [episode_idx, ...]}
   ```

2. **Update the index on write**:
   ```python
   def update_indexes(episode: dict, episode_idx: int, tool_context: ToolContext):
       # Type index
       type_idx = tool_context.state.get("user:type_index", {})
       type_key = episode["type"]
       type_idx.setdefault(type_key, []).append(episode_idx)
       tool_context.state["user:type_index"] = type_idx
   ```

3. **Query the type index**:
   ```python
   def get_episodes_by_type(episode_type: str, tool_context: ToolContext) -> list:
       type_idx = tool_context.state.get("user:type_index", {})
       episode_idxs = type_idx.get(episode_type, [])
       log = tool_context.state.get("user:episode_log", [])
       return [log[i] for i in episode_idxs if i < len(log)]
   ```

4. **Rebuild indexes** when the episode log is rebuilt after eviction:
   ```python
   def rebuild_type_index(log: list, tool_context: ToolContext):
       index = {}
       for i, episode in enumerate(log):
           index.setdefault(episode.get("type", "unknown"), []).append(i)
       tool_context.state["user:type_index"] = index
   ```

5. **Query keyword index** for text search:
   ```python
   def keyword_search(keyword: str, tool_context: ToolContext) -> list:
       kw_idx = tool_context.state.get("user:keyword_index", {})
       return kw_idx.get(keyword.lower(), [])
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "episodic",
  "content": "index_type=type|key=booking_completed|episode_idx=25",
  "metadata": {"index_action": "insert", "index_type": "type_index"},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "user:type_index:booking_completed",
  "memory_retrieved": [
    {"episode_idx": 25, "type": "booking_completed", "content": "JFK→LHR BA178"}
  ],
  "memory_injected": False
}
```

## Error Handling

- **Index out of sync** (episode evicted but index still holds its idx): Rebuild
  indexes after every eviction cycle. Run `rebuild_type_index()` after FIFO eviction.
- **Index key collision** (two episodes with the same type): Indexes hold lists
  of indices — multiple values per key are expected. No collision issue.
- **Missing index key** (new episode type not in type_index): Use
  `setdefault(type_key, [])` to handle new types automatically.
- **Index too large** (keyword index with thousands of unique keywords): Limit
  keyword index to top-K high-frequency keywords (> 3 occurrences).

## Examples

### Example 1 — Full index management system

```python
import json
import time
from typing import Optional
from google.adk.tools import ToolContext

def add_indexed_episode(episode_type: str, content: str, tool_context: ToolContext,
                          keywords: Optional[list] = None, metadata: Optional[dict] = None) -> dict:
    """Adds an episode to the log and updates all indexes atomically.

    Args:
        episode_type (str): Episode category for type index.
        content (str): Episode content for keyword index.
        tool_context (ToolContext): ADK tool context.
        keywords (list, optional): Keywords to index.
        metadata (dict, optional): Additional metadata.
    Returns:
        dict: Contains 'episode_idx' (int), 'indexes_updated' (list).
    """
    episode = {"type": episode_type, "content": content,
               "timestamp": time.time(), "session_id": tool_context.session_id,
               "metadata": metadata or {}}
    raw = tool_context.state.get("user:episode_log", [])
    log = json.loads(raw) if isinstance(raw, str) else (raw if isinstance(raw, list) else [])
    episode_idx = len(log)
    log.append(episode)
    tool_context.state["user:episode_log"] = log

    updated_indexes = []

    # Update type index
    type_idx = tool_context.state.get("user:type_index", {})
    type_idx.setdefault(episode_type, []).append(episode_idx)
    tool_context.state["user:type_index"] = type_idx
    updated_indexes.append("type_index")

    # Update date index
    date_key = time.strftime("%Y-%m-%d")
    date_idx = tool_context.state.get("user:date_index", {})
    date_idx.setdefault(date_key, []).append(episode_idx)
    tool_context.state["user:date_index"] = date_idx
    updated_indexes.append("date_index")

    # Update keyword index
    if keywords:
        kw_idx = tool_context.state.get("user:keyword_index", {})
        for kw in keywords:
            kw_idx.setdefault(kw.lower(), []).append(episode_idx)
        tool_context.state["user:keyword_index"] = kw_idx
        updated_indexes.append("keyword_index")

    return {"episode_idx": episode_idx, "indexes_updated": updated_indexes}


def query_by_type(episode_type: str, limit: int = 10, tool_context: ToolContext = None) -> dict:
    """Retrieves episodes of a specific type using the type index for O(1) lookup."""
    type_idx = tool_context.state.get("user:type_index", {})
    idxs = type_idx.get(episode_type, [])[-limit:]  # Most recent N entries
    log = tool_context.state.get("user:episode_log", [])
    log_list = json.loads(log) if isinstance(log, str) else (log if isinstance(log, list) else [])
    episodes = [log_list[i] for i in idxs if i < len(log_list)]
    return {"episodes": episodes, "total": len(episodes), "type_queried": episode_type}
```

## Edge Cases

- **Index rebuild on startup**: If the index is missing (e.g., first session after
  code change), rebuild all indexes from the episode log on `before_agent_callback`.
- **Compound index queries** (type=X AND date=Y): Intersect the index result sets:
  `set(type_idx[X]) & set(date_idx[Y])`.
- **Very high-cardinality keyword index** (thousands of unique keywords): Prune
  keywords with < 2 occurrences. Store the index in a more efficient format.

## Integration Notes

- **`google-adk-episodic-memory`**: Episode logs are the primary data structure
  indexed by this skill.
- **`google-adk-memory-eviction-strategies`**: After eviction, call
  `rebuild_type_index()` to keep indexes consistent.
- **`google-adk-memory-retrieval-strategies`**: Type-based and date-based retrieval
  use indexes built by this skill.
- **`google-adk-vector-memory`**: Vector indexing is a specialized form of memory
  indexing — this skill covers non-vector structural indexes.
