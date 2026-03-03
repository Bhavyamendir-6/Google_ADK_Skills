---
name: redundancy-elimination
description: >
  Use this skill when identifying and eliminating redundant computations,
  duplicate tool calls, repeated retrievals, and overlapping logic in a Google
  ADK agent pipeline. Covers deduplication of tool invocations within a turn,
  detecting repeated state operations, merging redundant sub-agents, removing
  unnecessary pipeline stages, and applying memoization for idempotent operations.
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

This skill identifies and eliminates redundant computation in Google ADK agent
pipelines — any repeated work that produces the same result as a prior computation
in the same request or session. Redundancy manifests as: the agent calling the same
tool multiple times with identical arguments in a single turn, multiple `ParallelAgent`
sub-agents performing overlapping retrievals, redundant state reads that recompute
the same derived value, and duplicate pipelines across different entry points.
Eliminating redundancy reduces latency, token usage, cost, and API call volume
simultaneously — often with zero quality impact.

## When to Use

- When profiling reveals the same tool is called multiple times per turn with
  identical arguments.
- When two sub-agents in a `ParallelAgent` query the same data source for overlapping data.
- When the same state value is recomputed from raw data on every turn instead
  of being cached.
- When multiple pipeline paths perform the same preprocessing steps independently.
- When audit logs show duplicate API calls to external services within a single session.

## When NOT to Use

- Do not deduplicate tool calls that require fresh data on each invocation (e.g.,
  real-time stock prices, live inventory).
- Do not merge sub-agents that are conceptually distinct even if they sometimes
  query overlapping data.
- Do not apply memoization to tools with side effects (state changes, payment
  processing).

## Google ADK Context

- **Within-turn deduplication**: A `tool_context`-level call registry: record
  `(tool_name, args_hash)` when a tool is first called. On subsequent calls with
  the same inputs, return the cached result immediately.
- **`temp:` prefix for within-turn memoization**: `temp:memo:{tool}:{args_hash}` —
  auto-cleared after each turn. Ideal for deduplication within a single multi-step
  reasoning turn.
- **Tool call audit log**: Record all tool calls with timestamps and arguments in
  `tool_context.state["temp:tool_call_log"]` to detect redundancy post-hoc.
- **`ParallelAgent` overlap**: If multiple sub-agents call `retrieve_context(query=X)`
  with the same query, they each trigger a separate vector search. Restructure to
  retrieve once in a preceding step and inject the shared result.
- **Idempotent operations**: Deterministic, read-only operations (lookup, search,
  retrieve) are safe to memoize. Mutating operations (booking, payment) must
  never be memoized.
- **State-derived value recomputation**: If the agent recomputes a value from
  session state on every turn (e.g., `sum(prices)`), compute it once and store
  the result in state.

## Capabilities

- Detects redundant same-turn tool calls via a within-turn call registry.
- Memoizes idempotent tool results in `temp:` state for within-turn reuse.
- Identifies and removes redundant parallel sub-agent overlaps.
- Audits tool call logs to surface redundancy patterns post-hoc.
- Merges overlapping preprocessing steps across pipeline paths.
- Eliminates repeated state derivations by caching derived values.

## Step-by-Step Instructions

1. **Audit tool calls for redundancy** — add a call log to all tool functions:
   ```python
   import hashlib, json, time
   def log_tool_call(tool_name: str, args: dict, tool_context: ToolContext):
       log = tool_context.state.get("temp:tool_call_log", [])
       log.append({"tool": tool_name, "args": args, "ts": time.time()})
       tool_context.state["temp:tool_call_log"] = log
   ```

2. **Detect duplicate calls** in the log:
   ```python
   def find_duplicates(call_log: list) -> list:
       seen = {}
       duplicates = []
       for call in call_log:
           key = f"{call['tool']}:{json.dumps(call['args'], sort_keys=True)}"
           if key in seen:
               duplicates.append(call)
           seen[key] = True
       return duplicates
   ```

3. **Implement within-turn memoization** for idempotent tools:
   ```python
   def memoized_search(query: str, tool_context: ToolContext) -> dict:
       memo_key = f"temp:memo:search:{hashlib.md5(query.encode()).hexdigest()[:10]}"
       cached = tool_context.state.get(memo_key)
       if cached is not None:
           return {**cached, "memoized": True}
       result = expensive_search(query)
       tool_context.state[memo_key] = result
       return result
   ```

4. **Eliminate parallel sub-agent overlap**: If multiple sub-agents need the same
   retrieved data, perform retrieval in a preceding sequential step and inject via
   `{retrieval_output?}`:
   ```python
   retrieval_step = LlmAgent(name="shared_retrieval", tools=[retrieve_context], output_key="shared_ctx")
   parallel_processors = ParallelAgent(sub_agents=[
       LlmAgent(instruction="Context: {shared_ctx?}\nAnalyze pricing..."),
       LlmAgent(instruction="Context: {shared_ctx?}\nAnalyze routing..."),
   ])
   pipeline = SequentialAgent(sub_agents=[retrieval_step, parallel_processors])
   ```

5. **Cache derived state values** instead of recomputing:
   ```python
   if "derived:total_price" not in tool_context.state:
       prices = [f["price"] for f in flights]
       tool_context.state["derived:total_price"] = sum(prices)
   ```

6. **Validate elimination correctness** via integration tests: confirm outputs
   match before and after redundancy elimination.

## Input Format

```python
{
  "operation": "redundancy_audit",
  "execution_time_ms": 6800,
  "token_usage": 16000,
  "memory_usage_mb": 128,
  "cost_estimate": 0.008,
  "retrieval_latency_ms": 320,
  "cache_available": False,
  "parallelizable": False,
  "context_size_tokens": 16000
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "within_turn_memoization + parallel_overlap_elimination + derived_value_caching",
  "performance_improvement": {
    "latency_reduction_ms": 2400,
    "token_reduction": 6500,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.0032,
    "throughput_improvement": 1.8
  }
}
```

## Error Handling

- **Memoized result returned for a different user** (user_id scope bug): Always
  include `tool_context.user_id` in the memo key for user-specific results:
  `f"temp:memo:{tool_context.user_id}:{hash}"`.
- **Memoized stale result** (underlying data changed mid-session): Only memoize
  results with a TTL. Include expiry check: `if time.time() < expires_at: return cached`.
- **Integration test catches output difference** after redundancy elimination:
  The eliminated call had a side effect or data dependency. Restore it. Re-audit.
- **Deduplication blocks a legitimate second call** (user asked the same question
  twice intentionally): Scope `temp:` memos to be cleared at the right boundaries.
  `temp:` clears per invocation — within-invocation dedup is safe.

## Examples

### Example 1 — Memoization decorator for ADK tool functions

```python
import hashlib
import json
import functools
import logging
from google.adk.tools import ToolContext

logger = logging.getLogger("memoize")

def adk_memoize(ttl_seconds: int = 0, scope: str = "turn"):
    """Decorator that memoizes an ADK tool function's result.

    Args:
        ttl_seconds (int): Cache TTL. 0 for within-turn only (auto-cleared).
        scope (str): 'turn' (temp:), 'session' (no prefix), 'user' (user:).
    """
    prefix = {"turn": "temp:memo", "session": "memo", "user": "user:memo"}.get(scope, "temp:memo")

    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, tool_context: ToolContext = None, **kwargs):
            all_args = {**kwargs}
            for i, a in enumerate(args):
                all_args[f"arg_{i}"] = str(a)
            arg_hash = hashlib.md5(json.dumps(all_args, sort_keys=True).encode()).hexdigest()[:10]
            cache_key = f"{prefix}:{fn.__name__}:{arg_hash}"
            expires_key = f"{cache_key}_exp"

            import time
            if (ttl_seconds == 0 or tool_context.state.get(expires_key, 0) > time.time()):
                cached = tool_context.state.get(cache_key)
                if cached is not None:
                    logger.debug(f"Memo HIT: {fn.__name__}")
                    return {**cached, "memoized": True} if isinstance(cached, dict) else cached

            result = fn(*args, tool_context=tool_context, **kwargs)
            tool_context.state[cache_key] = result
            if ttl_seconds > 0:
                tool_context.state[expires_key] = time.time() + ttl_seconds
            return result
        return wrapper
    return decorator


@adk_memoize(ttl_seconds=600, scope="session")
def search_flights(origin: str, destination: str, date: str, tool_context: ToolContext) -> dict:
    """Searches for flights. Results are memoized for 10 minutes within the session."""
    # ... actual search implementation
    return {"flights": [], "memoized": False}
```

### Example 2 — Shared retrieval step before parallel processing

```python
from google.adk.agents import LlmAgent, ParallelAgent, SequentialAgent

# BEFORE — each parallel sub-agent ran its own retrieval (3× vector searches)
# AFTER — shared retrieval runs once, injected via {policy_context?}

shared_retrieval = LlmAgent(
    name="shared_policy_retrieval",
    model="gemini-2.0-flash-lite",
    instruction="Retrieve all relevant policy context for: {user_query?}",
    tools=[retrieve_policy_context],
    output_key="policy_context",
)

price_analyzer = LlmAgent(
    name="price_analyzer",
    model="gemini-2.0-flash-lite",
    instruction="Policy: {policy_context?}\nAnalyze pricing options for {route?}",
    tools=[analyze_pricing],
    output_key="price_analysis",
)

route_planner = LlmAgent(
    name="route_planner",
    model="gemini-2.0-flash-lite",
    instruction="Policy: {policy_context?}\nPlan optimal route for {destination?}",
    tools=[plan_route],
    output_key="route_plan",
)

parallel_analysis = ParallelAgent(
    name="parallel_analysis",
    sub_agents=[price_analyzer, route_planner],
)

no_redundancy_pipeline = SequentialAgent(
    name="deduplicated_pipeline",
    sub_agents=[shared_retrieval, parallel_analysis],
)
```

## Edge Cases

- **ADK model calls same tool multiple times in one reasoning turn** (model bug):
  Implement `within_turn_call_count` tracking. After N calls to the same tool
  with same args, return a synthesis instruction: "The tool has already been called.
  Use the previous result."
- **Memoization saves wrong result** (race condition in async execution): `tool_context.state`
  is not thread-safe in `InMemorySessionService`. Use `DatabaseSessionService`'s
  transactional guarantees for concurrent session access.
- **Merged pipeline produces higher latency** (hidden sequential dependency):
  Profile the merged pipeline. Sometimes the overhead of sharing output is higher
  than the cost of duplication for very fast retrievals.

## Integration Notes

- **`google-adk-caching-strategies`**: Caching is the session/cross-session layer;
  redundancy elimination is the within-request layer.
- **`google-adk-execution-pipeline-optimization`**: Removing redundant pipeline
  stages is a specific form of execution pipeline optimization.
- **`google-adk-token-usage-optimization`**: Eliminating redundant tool calls
  also eliminates the tokens consumed by those calls in the reasoning loop.
- **`google-adk-latency-optimization`**: Removing duplicate slow calls is the
  most impactful latency reduction without quality trade-offs.
