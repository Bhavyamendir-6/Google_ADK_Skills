---
name: memory-usage-optimization
description: >
  Use this skill when reducing in-process memory consumption of a Google ADK
  agent deployment. Covers managing Python object lifecycle within tool functions,
  keeping tool_context.state values compact, limiting session history size,
  optimizing vector store memory footprint, and monitoring memory usage in
  production ADK deployments.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: performance-optimization
---

## Purpose

This skill reduces in-process memory consumption of Google ADK agent deployments.
Memory usage is primarily driven by: large `tool_context.state` values held in
the session service, large Python objects created inside tool functions, vector
store index sizes held in memory, and the accumulated conversation history in
long-running sessions. Uncontrolled memory growth leads to OOM (out-of-memory)
kills, performance degradation under GC pressure, and excessive hosting costs.
This skill audits each memory source and applies ADK-native reduction strategies.

## When to Use

- When the ADK service process is killed by the OS due to OOM errors.
- When heap memory usage grows unboundedly across many sessions.
- When tool functions create large intermediate objects (large DataFrames, raw API responses).
- When `tool_context.state` values grow beyond 64KB per entry.
- When an in-memory ChromaDB vector store holds too many embeddings per process.
- When session history accumulation inflates session objects in memory.

## When NOT to Use

- Do not apply memory optimization prematurely without first profiling actual usage.
- Do not remove session state values needed for agent functionality.
- Do not shrink vector store chunk sizes below semantic coherence thresholds.

## Google ADK Context

- **`tool_context.state`**: The session service holds state in memory
  (`InMemorySessionService`) or serializes to a database. Large state values
  consume memory per active session.
- **Tool function objects**: Large intermediate Python objects (DataFrames, JSON
  dicts with 1000+ items) created inside tool functions should be explicitly freed.
- **Session history events**: Each session accumulates `Event` objects in memory.
  `InMemorySessionService` holds all events for all active sessions.
- **Vector store memory**: ChromaDB in-memory collections hold embeddings in RAM.
  For large corpora, use a persistent client or external service.
- **Python GC**: ADK tool functions run in the same Python process. Large objects
  in tool function scope are GC'd when the function returns — but only if no
  reference is held elsewhere.
- **Memory monitoring**: Use `tracemalloc`, `psutil`, or cloud metrics to track
  per-process memory usage.

## Capabilities

- Reduces `tool_context.state` value sizes by compressing stored content.
- Frees large tool function objects explicitly after use.
- Limits session history size via summarization and eviction.
- Moves large vector stores from in-memory to persistent disk-backed clients.
- Monitors per-process memory usage from within the ADK service.
- Implements session-level memory budgets.

## Step-by-Step Instructions

1. **Profile memory usage** under load:
   ```python
   import psutil, os
   process = psutil.Process(os.getpid())
   mem_mb = process.memory_info().rss / 1024 / 1024
   ```

2. **Identify large tool_context.state values**:
   ```python
   for key, val in tool_context.state.items():
       size = len(str(val))
       if size > 10_000:
           logger.warning(f"Large state value: {key} = {size} chars")
   ```

3. **Compress state values** before storing:
   ```python
   # Store only top-5 results, not all 200
   results = search_api(query)
   tool_context.state["results"] = [r[:200] for r in results[:5]]
   ```

4. **Explicitly delete large intermediate objects** in tools:
   ```python
   def process_large_data(data_url: str, tool_context: ToolContext) -> dict:
       raw_data = fetch_large_dataset(data_url)  # 50MB DataFrame
       summary = raw_data.describe().to_dict()   # 1KB summary
       del raw_data  # Explicitly free the large object
       return {"summary": summary}
   ```

5. **Switch ChromaDB from in-memory to persistent**:
   ```python
   import chromadb
   client = chromadb.PersistentClient(path="./chroma_db")  # Not EphemeralClient
   ```

6. **Apply session history limits** (see `memory-summarization` skill):
   Summarize after 20 turns to prevent history event accumulation.

7. **Switch from `InMemorySessionService` to `DatabaseSessionService`**:
   Database backend serializes state to disk rather than holding it in RAM.

8. **Monitor memory per invocation** and alert on growth > threshold.

## Input Format

```python
{
  "operation": "large_data_processing",
  "execution_time_ms": 2100,
  "token_usage": 4200,
  "memory_usage_mb": 512,
  "cost_estimate": 0.0021,
  "retrieval_latency_ms": 180,
  "cache_available": False,
  "parallelizable": False,
  "context_size_tokens": 4200
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "state_compression + large_object_deletion + persistent_vector_store",
  "performance_improvement": {
    "latency_reduction_ms": 0,
    "token_reduction": 0,
    "memory_reduction_mb": 380,
    "cost_reduction": 0.0,
    "throughput_improvement": 1.4
  }
}
```

## Error Handling

- **Deleting large object while still referenced**: Ensure no other variable holds
  a reference. Use `with` blocks or `contextlib.closing` for resource management.
- **`DatabaseSessionService` uses more disk than expected**: Set session TTL to
  expire old sessions. Implement a periodic cleanup job.
- **ChromaDB persistent client fails to write** (disk full): Monitor disk usage
  alongside RAM. Implement eviction when disk usage > 80%.
- **Memory leak in long-running session**: Use `tracemalloc` to identify the
  growing object type. Common culprits: global dicts accumulating session references, logging handlers.

## Examples

### Example 1 — Memory-safe tool function pattern

```python
import gc
import logging
from google.adk.tools import ToolContext

logger = logging.getLogger("memory_opt")

def analyze_flight_data(data_url: str, tool_context: ToolContext) -> dict:
    """Analyzes flight data with explicit memory management.

    Args:
        data_url (str): URL of the flight dataset to analyze.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'summary' (dict) with compact analysis results.
    """
    # Fetch large dataset
    raw = fetch_flight_data(data_url)  # May be 10MB+
    try:
        # Process and extract only the summary we need
        summary = {
            "route_count": len(raw.get("routes", [])),
            "avg_price": sum(r.get("price", 0) for r in raw.get("routes", [])) / max(len(raw.get("routes", [])), 1),
            "top_airlines": list(set(r.get("airline") for r in raw.get("routes", [])[:10])),
        }
        # Store compact summary, not raw data
        tool_context.state["flight_analysis"] = summary
        return summary
    finally:
        del raw  # Explicitly free the large object
        gc.collect()  # Trigger GC for large object

def fetch_flight_data(url: str) -> dict:
    return {}  # Placeholder
```

### Example 2 — Per-session memory budget enforcer

```python
import logging
import sys
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content, Part

logger = logging.getLogger("memory_budget")
MAX_STATE_VALUE_CHARS = 8_000

def enforce_memory_budget(callback_context: CallbackContext) -> Optional[Content]:
    """Enforces a per-session memory budget by removing oversized state entries."""
    oversized = []
    for key, val in list(callback_context.state.items()):
        val_size = len(str(val))
        if val_size > MAX_STATE_VALUE_CHARS:
            oversized.append((key, val_size))
            # Remove and replace with a truncated placeholder
            del callback_context.state[key]
            callback_context.state[key] = f"[{val_size} chars — compressed. Key: {key}]"
            logger.warning(f"State value truncated: {key} was {val_size} chars.")
    return None
```

## Edge Cases

- **Large user message in session history**: The user's message is part of the
  session event log. Long user messages (e.g., pasted document) inflate memory.
  Add a pre-processing step to summarize user inputs > 2000 chars before passing
  to the agent.
- **Concurrent sessions** (1000 simultaneous users): With `InMemorySessionService`,
  each session's state is held in RAM. Switch to `DatabaseSessionService` to
  serialize state to disk.
- **Vector store grows without bound**: Implement max-entries-per-user limit with
  LRU eviction when the limit is reached.

## Integration Notes

- **`google-adk-memory-eviction-strategies`**: Eviction keeps the episode log
  and state values within memory budget.
- **`google-adk-memory-storage-backends`**: `DatabaseSessionService` offloads
  session state from RAM to disk.
- **`google-adk-memory-compression`**: Compresses state values before storage
  to reduce both state size and memory footprint.
- **`google-adk-resource-utilization-optimization`**: Covers the broader resource
  (CPU, memory, network) optimization strategy.
