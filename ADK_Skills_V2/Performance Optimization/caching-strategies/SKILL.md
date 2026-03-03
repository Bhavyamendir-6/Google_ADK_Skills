---
name: caching-strategies
description: >
  Use this skill when implementing caching for Google ADK agent tool results,
  model outputs, retrieval results, and session state computations to eliminate
  redundant expensive operations. Covers TTL-based key-value caching in
  tool_context.state, content-hash-based cache keying, cache invalidation
  patterns, distributed cache integration, and monitoring cache hit rates.
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

This skill implements caching for Google ADK agents to eliminate costly
recomputation of tool results, retrieval results, and model outputs. Caching
stores the result of an expensive operation keyed by its inputs; on subsequent
identical requests, the cached result is returned immediately without re-executing
the operation. In ADK, the primary caching layer is `tool_context.state` with
TTL enforcement — lightweight, zero-dependency caching available in every tool
function. For production deployments, this skill also covers external distributed
caches (Redis) and semantic caches (embedding-based similarity matching of queries).

## When to Use

- When the same search query or API call is repeated across turns or sessions
  (e.g., daily flight schedules, weather, exchange rates).
- When vector retrieval results for similar queries are reused within a session.
- When model outputs for deterministic, frequently-asked questions can be cached.
- When tool API costs are significant and caching would reduce API calls.
- When latency profiling reveals a tool call as the dominant bottleneck.

## When NOT to Use

- Do not cache personalized results (user-specific pricing, seat availability)
  that must be fresh on every request.
- Do not cache results with very short validity windows (< 60 seconds) — the
  overhead of cache management exceeds the benefit.
- Do not return a stale cached result for safety-critical data (medical dosages,
  financial transactions).

## Google ADK Context

- **`tool_context.state` caching**: Store `{cache_key: result, cache_key_expires_at: timestamp}`.
  Read: check `_expires_at` before returning. Write: set both result and expiry.
- **`temp:` prefix caching**: Within-turn cache (auto-cleared after each turn).
  Use for expensive within-turn computations reused by multiple tools.
- **Session-scoped cache**: Standard `tool_context.state` keys. Available across
  turns in the same session.
- **`user:` prefix caching**: Cross-session cache for per-user data (e.g., user
  profile, historical preferences).
- **`app:` prefix caching**: Application-wide cache for global data (airport codes,
  currency rates). Shared across all users.
- **Cache key design**: `f"cache:{operation}:{hash(inputs)}"` for reliable uniqueness.
- **External Redis cache**: For multi-instance deployments where `tool_context.state`
  is not shared across replicas.
- **Semantic cache**: Match new queries against cached queries by embedding similarity
  (see `google-adk-retrieval-optimization`).

## Capabilities

- Implements TTL-based key-value caching in `tool_context.state`.
- Designs reliable cache keys using content hashing.
- Implements cache invalidation on data mutations.
- Monitors cache hit rate and miss rate from state analytics.
- Integrates with Redis for multi-instance deployments.
- Implements semantic caching for NL query similarity matching.
- Supports three cache scopes: within-turn, session, cross-session.

## Step-by-Step Instructions

1. **Identify cacheable operations**: Tool calls that are deterministic, expensive,
   and called with repeated inputs within a session or across turns.

2. **Design the cache key**: `f"cache:{tool_name}:{hashlib.md5(str(inputs).encode()).hexdigest()[:12]}"`

3. **Read from cache first**:
   ```python
   import time, hashlib, json
   cache_key = f"cache:search:{hashlib.md5(query.lower().encode()).hexdigest()[:12]}"
   expires_key = f"{cache_key}_expires_at"
   if tool_context.state.get(expires_key, 0) > time.time():
       return {"result": tool_context.state[cache_key], "cached": True}
   ```

4. **Execute and write to cache**:
   ```python
   result = expensive_search(query)
   tool_context.state[cache_key] = result
   tool_context.state[expires_key] = time.time() + 300  # 5-minute TTL
   return {"result": result, "cached": False}
   ```

5. **Choose the right scope**:
   - Within-turn: `temp:cache:...`
   - Session: `cache:...` (no prefix for session scope)
   - Cross-session per-user: `user:cache:...`
   - Global: `app:cache:...`

6. **Implement cache invalidation** when data changes:
   ```python
   def invalidate_route_cache(origin: str, destination: str, tool_context: ToolContext):
       keys_to_delete = [k for k in tool_context.state if f"cache:search:{origin}_{destination}" in k]
       for k in keys_to_delete:
           del tool_context.state[k]
   ```

7. **Monitor hit rate** by incrementing counters:
   ```python
   hits = int(tool_context.state.get("cache_hits", 0)) + 1
   tool_context.state["cache_hits"] = hits
   ```

## Input Format

```python
{
  "operation": "flight_search",
  "execution_time_ms": 1800,
  "token_usage": 0,
  "memory_usage_mb": 8,
  "cost_estimate": 0.0,
  "retrieval_latency_ms": 0,
  "cache_available": True,
  "parallelizable": False,
  "context_size_tokens": 0
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "ttl_state_cache",
  "performance_improvement": {
    "latency_reduction_ms": 1750,
    "token_reduction": 0,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.0015,
    "throughput_improvement": 8.5
  }
}
```

## Error Handling

- **Cache miss after supposed hit** (TTL check bug): Always use `> time.time()` not
  `>= time.time()`. Add a 1-second buffer: `time.time() + TTL - 1`.
- **Stale cache returned** (data changed upstream): Implement manual invalidation
  tool. Add a cache version key: `app:cache_version` incremented on data updates.
- **Cache key collision** (two operations produce the same key): Use the full
  tool name in the key prefix. Use `sha256` instead of `md5` for lower collision probability.
- **Cache grows unboundedly** (no eviction): Apply size-triggered eviction.
  Delete cache keys in order of expiry when total key count > MAX_CACHE_KEYS.
- **Redis connection failure**: Fall back to `tool_context.state` cache. Never fail
  the tool call because caching is unavailable.

## Examples

### Example 1 — Multi-scope caching tool

```python
import time
import hashlib
import json
import logging
from typing import Optional, Any
from google.adk.tools import ToolContext

logger = logging.getLogger("cache")

def cached_tool_call(
    tool_name: str,
    tool_args: dict,
    tool_fn,
    tool_context: ToolContext,
    ttl_seconds: int = 300,
    scope: str = "session",
) -> dict:
    """Generic caching wrapper for any ADK tool function.

    Args:
        tool_name (str): Tool identifier for cache key namespace.
        tool_args (dict): Tool arguments (used for cache key generation).
        tool_fn: The actual tool function to call on cache miss.
        tool_context (ToolContext): ADK tool context.
        ttl_seconds (int): Cache TTL in seconds.
        scope (str): Cache scope — 'turn', 'session', 'user', 'global'.
    Returns:
        dict: Tool result (from cache or fresh execution).
    """
    arg_hash = hashlib.md5(json.dumps(tool_args, sort_keys=True).encode()).hexdigest()[:12]
    scope_prefix = {"turn": "temp:cache", "session": "cache", "user": "user:cache", "global": "app:cache"}.get(scope, "cache")
    cache_key = f"{scope_prefix}:{tool_name}:{arg_hash}"
    expires_key = f"{cache_key}_exp"

    # Check cache
    if tool_context.state.get(expires_key, 0) > time.time():
        logger.debug(f"Cache HIT: {cache_key}")
        hit_count = int(tool_context.state.get("cache_hits", 0)) + 1
        tool_context.state["cache_hits"] = hit_count
        return {**tool_context.state[cache_key], "cached": True}

    # Cache miss — execute the tool
    logger.debug(f"Cache MISS: {cache_key}")
    result = tool_fn(**tool_args, tool_context=tool_context)

    # Store result and TTL
    tool_context.state[cache_key] = result
    tool_context.state[expires_key] = time.time() + ttl_seconds
    miss_count = int(tool_context.state.get("cache_misses", 0)) + 1
    tool_context.state["cache_misses"] = miss_count
    return {**result, "cached": False}
```

### Example 2 — App-scoped global cache for airport codes

```python
from google.adk.tools import ToolContext

def get_airport_info(airport_code: str, tool_context: ToolContext) -> dict:
    """Gets airport information with app-scoped (global) caching.

    Args:
        airport_code (str): IATA airport code.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'name' (str), 'city' (str), 'cached' (bool).
    """
    cache_key = f"app:cache:airport:{airport_code.upper()}"
    # Airport data changes rarely — 24 hour TTL
    if tool_context.state.get(f"{cache_key}_exp", 0) > __import__("time").time():
        data = tool_context.state.get(cache_key, {})
        return {**data, "cached": True}

    # Fetch from external source
    data = fetch_airport_data(airport_code)  # External API call
    tool_context.state[cache_key] = data
    tool_context.state[f"{cache_key}_exp"] = __import__("time").time() + 86400  # 24h TTL
    return {**data, "cached": False}

def fetch_airport_data(code: str) -> dict:
    return {"name": f"Airport {code}", "city": "Unknown"}  # Placeholder
```

## Edge Cases

- **Same cache key for different users** (shared `app:` cache with user-specific data):
  Only use `app:` cache for truly global, non-personalized data. Use `user:cache:` for
  per-user cached data.
- **Cache poisoning** (malformed tool result stored): Validate tool result before
  writing to cache. Don't cache error responses.
- **Very high TTL** (cached data outlives its validity): Model TTL based on
  the upstream data's update frequency. Use webhook-based invalidation for
  real-time upstream changes.

## Integration Notes

- **`google-adk-latency-optimization`**: Caching is the highest-impact latency
  optimization for repeated queries.
- **`google-adk-cost-optimization`**: Cached results cost $0 — eliminating
  redundant external API and model inference costs.
- **`google-adk-retrieval-optimization`**: Caching retrieval results avoids
  expensive repeated vector searches.
- **`google-adk-memory-eviction-strategies`**: Cache key eviction prevents
  the cache from consuming unlimited state storage.
