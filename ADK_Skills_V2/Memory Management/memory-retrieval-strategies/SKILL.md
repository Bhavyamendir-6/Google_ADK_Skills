---
name: memory-retrieval-strategies
description: >
  Use this skill when designing strategies for retrieving relevant memory entries
  from a Google ADK agent's memory stores. Covers exact key-value retrieval from
  tool_context.state, time-based recency retrieval, type-filtered episode retrieval,
  vector similarity retrieval, and hybrid retrieval combining multiple strategies
  for optimal recall and precision.
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

This skill designs and implements memory retrieval strategies for Google ADK
agents. Different memory retrieval needs require different strategies: exact-key
retrieval for known state values, recency-based retrieval for recent episodes,
type-filtered retrieval for specific event categories, and semantic similarity
retrieval for "find what's relevant to this query." This skill selects and
combines the right strategy per use case, implements retrieval tool functions,
and ensures retrieved memories are properly validated and injected into agent context.

## When to Use

- When designing how the agent retrieves relevant memories before answering a query.
- When existing key-value lookup is insufficient and semantic retrieval is needed.
- When implementing hybrid retrieval (combine recency + similarity for best results).
- When the agent must choose between multiple memory retrieval strategies at runtime.
- When retrieval quality is poor and a strategy upgrade is needed.

## When NOT to Use

- Do not use complex retrieval strategies for simple session state reads — `state.get(key)` is sufficient.
- Do not use vector similarity retrieval when exact matching is needed (e.g., order_id lookup).
- Do not retrieve more memories than needed — inject only the top-k most relevant.

## Google ADK Context

- **Exact retrieval**: `tool_context.state.get("key")` — O(1) lookup for known keys.
- **Recency retrieval**: Sort episode log by timestamp descending; return top-N recent entries.
- **Type-filtered retrieval**: Filter episode log by `episode["type"]` field.
- **Session-scoped retrieval**: All `tool_context.state` keys without prefix.
- **User-scoped retrieval**: All `user:` prefixed keys for the current user.
- **Vector similarity retrieval**: Query a vector store with an embedding of the current query.
- **Hybrid retrieval**: Combine recency score + similarity score for a composite ranking.
- **`tool_context.user_id`**: Required for user-namespaced retrieval.
- **`tool_context.session_id`**: Required for session-scoped filtering.

## Capabilities

- Implements exact-key state retrieval from `tool_context.state`.
- Implements recency-based episode retrieval sorted by timestamp.
- Implements type-filtered memory retrieval.
- Implements vector similarity retrieval with top-k results.
- Implements hybrid retrieval combining recency and similarity signals.
- Validates and ranks retrieved memories by relevance.
- Injects retrieved memories into agent context.

## Step-by-Step Instructions

1. **Select retrieval strategy** based on the retrieval need:
   - Known key → exact retrieval
   - Recent history → recency retrieval
   - Specific event category → type-filtered retrieval
   - Semantic relevance → vector similarity retrieval
   - Best general recall → hybrid retrieval

2. **Implement the chosen strategy as a tool function**:
   ```python
   def retrieve_memories(query: str, strategy: str, tool_context: ToolContext) -> dict:
       if strategy == "exact":
           return {"result": tool_context.state.get(query)}
       elif strategy == "recency":
           return _recency_retrieval(tool_context, limit=5)
       elif strategy == "vector":
           return _vector_retrieval(query, tool_context, k=5)
       elif strategy == "hybrid":
           return _hybrid_retrieval(query, tool_context)
   ```

3. **Validate retrieved memories** — filter out nulls, corrupted entries, or expired data.

4. **Rank by relevance** (for hybrid strategy): `score = 0.6 * similarity + 0.4 * recency_score`.

5. **Inject top-k retrieved memories** into state for instruction injection:
   ```python
   tool_context.state["temp:retrieved_memories"] = format_memories(top_k_results)
   ```

6. **Evaluate retrieval quality** using held-out test cases and measuring recall@k.

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "episodic",
  "content": "query=find flights I've booked before",
  "metadata": {"strategy": "hybrid", "k": 5},
  "embedding": [0.12, 0.45, -0.33]
}
```

## Output Format

```python
{
  "memory_stored": False,
  "memory_id": None,
  "memory_retrieved": [
    {"content": "Booked JFK→LHR BA178", "score": 0.92, "type": "booking_completed"},
    {"content": "Booked CDG→JFK AF007", "score": 0.85, "type": "booking_completed"}
  ],
  "memory_injected": True
}
```

## Error Handling

- **No memories found** (empty retrieval): Return empty list. Do not inject empty
  string into instruction — use `{key?}` which resolves to nothing.
- **Vector store unavailable**: Fall back to recency retrieval from episode log in state.
- **Score threshold too high** (no results meet 0.80 threshold): Lower threshold
  to 0.65 for sparse memory domains.
- **Retrieval latency too high** (> 500ms): Cache recent retrieval results in
  `temp:` state for the current invocation.
- **Corrupted memory entry** (missing required fields): Skip it with a warning log.

## Examples

### Example 1 — Multi-strategy retrieval tool

```python
import json
import time
from typing import Optional
from google.adk.tools import ToolContext

def retrieve_user_memories(
    query: str,
    strategy: str = "hybrid",
    k: int = 5,
    tool_context: ToolContext = None
) -> dict:
    """Retrieves relevant user memories using the specified strategy.

    Args:
        query (str): Query to retrieve memories for.
        strategy (str): One of: exact, recency, vector, hybrid.
        k (int): Max number of results to return.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'memories' (list), 'strategy_used' (str).
    """
    if strategy == "recency":
        return _recency_retrieval(tool_context, limit=k)
    elif strategy == "vector":
        return _vector_retrieval(query, tool_context, k=k)
    elif strategy == "hybrid":
        recency = _recency_retrieval(tool_context, limit=k * 2)
        vector = _vector_retrieval(query, tool_context, k=k * 2)
        return _merge_hybrid_results(recency, vector, k=k)
    return {"memories": [], "strategy_used": strategy}


def _recency_retrieval(tool_context: ToolContext, limit: int = 5) -> dict:
    raw = tool_context.state.get("user:episode_log", [])
    log = json.loads(raw) if isinstance(raw, str) else (raw if isinstance(raw, list) else [])
    recent = sorted(log, key=lambda e: e.get("timestamp", 0), reverse=True)[:limit]
    return {"memories": recent, "strategy_used": "recency"}


def _vector_retrieval(query: str, tool_context: ToolContext, k: int = 5) -> dict:
    # Placeholder — in practice calls the vector store (ChromaDB, Vertex AI)
    return {"memories": [], "strategy_used": "vector"}


def _merge_hybrid_results(recency_result: dict, vector_result: dict, k: int = 5) -> dict:
    """Merges recency and vector results using a weighted composite score."""
    entries = {}
    now = time.time()
    for mem in recency_result.get("memories", []):
        age_hours = (now - mem.get("timestamp", now)) / 3600
        recency_score = max(0.0, 1.0 - age_hours / 720)  # 720 hours = 30 days
        entries[mem.get("content", "")] = {"memory": mem, "recency": recency_score, "vector": 0.0}
    for mem in vector_result.get("memories", []):
        content = mem.get("content", "")
        if content in entries:
            entries[content]["vector"] = mem.get("score", 0.0)
        else:
            entries[content] = {"memory": mem, "recency": 0.0, "vector": mem.get("score", 0.0)}
    ranked = sorted(
        entries.values(),
        key=lambda e: 0.4 * e["recency"] + 0.6 * e["vector"],
        reverse=True
    )[:k]
    return {"memories": [e["memory"] for e in ranked], "strategy_used": "hybrid"}
```

## Edge Cases

- **Same memory retrieved by both recency and vector** (hybrid): Dedup by content
  hash before merging. Count it once with the highest composite score.
- **New user with no memory** (empty log): All strategies return empty. The agent
  proceeds without memory context — this is correct behavior.
- **Query is very short** (< 3 words): Short queries produce poor embeddings.
  Fall back to recency strategy for very short queries.
- **Retrieval returns too many irrelevant memories**: Lower k, raise the similarity
  threshold, or switch from hybrid to pure vector with a higher threshold.

## Integration Notes

- **`google-adk-vector-memory`**: Provides the vector store backend used by
  the vector and hybrid retrieval strategies.
- **`google-adk-episodic-memory`**: The episode log is the primary source for
  recency and type-filtered retrieval strategies.
- **`google-adk-retrieval-augmented-generation`**: RAG pipelines wrap vector
  retrieval strategies with chunking, embedding, and context injection.
- **`google-adk-context-window-management`**: Limits how many retrieved memories
  can be injected into the context window per turn.
