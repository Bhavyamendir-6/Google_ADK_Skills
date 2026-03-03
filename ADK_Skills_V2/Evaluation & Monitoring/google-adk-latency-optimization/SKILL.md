---
name: google-adk-latency-optimization
description: >
  Use this skill when a Google ADK agent's response latency is too high and
  needs systematic reduction. Covers identifying latency bottlenecks in tool
  calls, LLM invocations, and session management; applying caching strategies,
  async parallelism, tool response streaming, and model selection optimizations
  to reduce end-to-end agent invocation time.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: evaluation-and-monitoring
---

## Purpose

This skill systematically reduces end-to-end latency for Google ADK agents.
It covers identifying the dominant latency contributors (LLM call, tool call,
session I/O, network round trips), and applying targeted optimizations including
tool result caching, async parallel tool execution, streaming responses, prompt
compression, model downgrade for simple tasks, and session service backend
optimization. It is applied after `google-adk-performance-benchmarking`
identifies the specific bottleneck.

## When to Use

- When P95 end-to-end invocation latency exceeds the SLA (e.g., > 3 seconds).
- When a specific tool call is identified as the latency bottleneck via tracing.
- When the LLM model is over-sized for the task complexity (e.g., using
  `gemini-2.0-pro` for a simple classification task).
- When multiple independent tool calls are executed sequentially and could be
  parallelized.
- When session I/O on `DatabaseSessionService` is slow due to unindexed queries.

## When NOT to Use

- Do not optimize latency at the expense of correctness — validate with
  `google-adk-automated-evaluation` after each optimization.
- Do not cache tool results that must be fresh per invocation (e.g., real-time
  stock prices, live sensor data).
- Do not reduce model size (`gemini-flash` vs `gemini-pro`) without running
  evaluation to confirm quality is maintained.

## Google ADK Context

- **LLM Call Latency**: The dominant latency source in most ADK agents (400ms–3s).
  Reduce via: smaller model, shorter prompts, streaming, response caching.
- **Tool Call Latency**: Secondary latency source. Reduce via: caching, async
  I/O, batching, parallelism using `ParallelAgent`.
- **`ParallelAgent`**: ADK built-in agent that executes multiple sub-agents
  concurrently. Use to parallelize independent tool calls.
- **`SequentialAgent`**: Executes agents in order. Replace with `ParallelAgent`
  for independent steps.
- **Streaming responses**: ADK `Runner.run()` yields events as they are produced.
  Stream the final response to the user incrementally to reduce perceived latency.
- **`tool_context.state` caching**: Cache expensive tool results in `session.state`
  keyed by query hash. Skip re-execution on cache hit.
- **`InMemorySessionService` vs `DatabaseSessionService`**: In-memory is much
  faster for session lookups. Use it in latency-critical paths if persistence
  is not required.
- **`before_agent_callback`**: Use to inject pre-computed context (cached
  summaries, pre-fetched data) to reduce LLM reasoning steps.

## Capabilities

- Identifies the latency bottleneck via tracing spans on tool calls and LLM
  invocations.
- Implements in-session caching of expensive tool results.
- Parallelizes independent tool calls using `ParallelAgent`.
- Reduces LLM prompt token count by compressing injected context.
- Implements streaming event consumption for lower perceived latency.
- Downgrades model from `gemini-pro` to `gemini-flash` for latency-sensitive tasks.
- Optimizes `DatabaseSessionService` queries with connection pooling.

## Step-by-Step Instructions

1. **Identify the bottleneck** using tracing (see `google-adk-tracing`):
   - LLM call > 80% of total time → optimize model/prompt.
   - Tool call > 50% of total time → cache or parallelize.
   - Session I/O > 20% of total time → switch to faster session backend.

2. **Cache expensive tool results** in `tool_context.state`:
   ```python
   import hashlib
   def expensive_tool(query: str, tool_context: ToolContext) -> dict:
       cache_key = f"cache:{hashlib.md5(query.encode()).hexdigest()}"
       cached = tool_context.state.get(cache_key)
       if cached:
           return cached
       result = perform_expensive_operation(query)
       tool_context.state[cache_key] = result
       return result
   ```

3. **Parallelize independent tool calls** using `ParallelAgent`:
   ```python
   from google.adk.agents import ParallelAgent, LlmAgent
   parallel = ParallelAgent(
       name="parallel_data_fetch",
       sub_agents=[market_data_agent, news_agent, weather_agent],
   )
   ```

4. **Switch to a faster model** for latency-sensitive simple tasks:
   ```python
   # Before: gemini-2.0-pro (slow, expensive)
   # After:  gemini-2.0-flash (fast, cost-effective)
   agent = LlmAgent(name="classifier", model="gemini-2.0-flash", ...)
   ```

5. **Compress injected context** to reduce LLM input token count:
   ```python
   def compressed_provider(ctx: ReadonlyContext) -> str:
       summary = ctx.state.get("conversation_summary", "")[:300]  # Cap at 300 chars
       return f"Context: {summary}\n\nTask: {ctx.state.get('task', '')}"
   ```

6. **Stream responses** for lower perceived latency:
   ```python
   for event in runner.run(user_id=uid, session_id=sid, new_message=msg):
       if event.content and event.content.parts:
           for part in event.content.parts:
               if hasattr(part, "text") and part.text:
                   print(part.text, end="", flush=True)  # Stream partial response
   ```

7. **Validate latency improvement** by re-running benchmarks and confirming
   P95 is now within SLA. Validate quality with eval suite.

8. **Document the optimization** with before/after P50/P95/P99 numbers in
   `benchmarks/OPTIMIZATION_LOG.md`.

## Input Format

```python
{
  "agent_id": "my_agent",
  "session_id": "sess_abc123",
  "tool_name": "search_products",
  "execution_time_ms": 2800,
  "status": "success",
  "error": None,
  "metadata": {
    "bottleneck_identified": "tool_call",
    "sla_p95_ms": 1500,
    "current_p95_ms": 2800,
    "optimization_target": "cache_tool_results"
  }
}
```

## Output Format

```python
{
  "metric_recorded": True,
  "evaluation_result": "pass",
  "latency_ms": 320,
  "cost_estimate": 0.0,
  "issues_detected": [],
  "optimization_applied": "tool_result_cache",
  "before_p95_ms": 2800,
  "after_p95_ms": 320,
  "improvement_pct": 88.6,
  "cache_hit_rate": 0.72
}
```

## Error Handling

- **Cache invalidation needed** (stale cached result): Add a TTL to cache keys:
  ```python
  import time
  cache_key = f"cache:{hash_key}:ttl_{int(time.time() // 300)}"  # 5-min TTL
  ```
- **`ParallelAgent` sub-agent failure**: One sub-agent failure does not abort
  others. Check all sub-agent results for errors before processing.
- **Model downgrade reduces quality**: Roll back to the original model.
  A/B test on traffic before committing to downgrade.
- **Streaming causes display issues**: Only stream in terminal/chat interfaces
  that render partial text. Check the client's streaming support before enabling.
- **Session backend too slow**: Profile `get_session()` and `append_event()`.
  Switch to `InMemorySessionService` for read-heavy non-persistent paths.

## Examples

### Example 1 — TTL-cached tool result

```python
import hashlib
import time
import json
from google.adk.tools import ToolContext

CACHE_TTL_SECONDS = 300  # 5 minutes

def search_catalog(query: str, tool_context: ToolContext) -> dict:
    """Searches the product catalog with 5-minute result caching.

    Args:
        query (str): Search query.
        tool_context (ToolContext): ADK tool context with state caching.

    Returns:
        dict: Contains 'results' (list), 'cached' (bool).
    """
    query_hash = hashlib.md5(query.lower().encode()).hexdigest()
    ttl_slot = int(time.time() // CACHE_TTL_SECONDS)
    cache_key = f"temp:cache:search:{query_hash}:{ttl_slot}"

    cached = tool_context.state.get(cache_key)
    if cached:
        return {"results": json.loads(cached) if isinstance(cached, str) else cached, "cached": True}

    results = _do_search(query)
    tool_context.state[cache_key] = json.dumps(results) if not isinstance(results, str) else results
    return {"results": results, "cached": False}

def _do_search(query):
    return [{"id": 1, "name": "Example Product"}]  # Placeholder for real search
```

### Example 2 — Parallel data fetching with ParallelAgent

```python
from google.adk.agents import LlmAgent, ParallelAgent, SequentialAgent

market_agent = LlmAgent(name="market_data", model="gemini-2.0-flash",
                         instruction="Fetch market data for {symbol}.",
                         tools=[fetch_market_data])

news_agent = LlmAgent(name="news_fetcher", model="gemini-2.0-flash",
                       instruction="Fetch latest news for {symbol}.",
                       tools=[fetch_news])

# Parallel: both run concurrently
parallel_fetch = ParallelAgent(
    name="parallel_fetch",
    sub_agents=[market_agent, news_agent],
)

# Sequential: analysis after parallel fetch
analyzer = LlmAgent(name="analyzer", model="gemini-2.0-pro",
                     instruction="Analyze data from parallel_fetch.")

pipeline = SequentialAgent(
    name="analysis_pipeline",
    sub_agents=[parallel_fetch, analyzer],
)
```

### Example 3 — Streaming response consumption

```python
import asyncio
from google.adk.runners import Runner
from google.genai.types import Content, Part

async def stream_agent_response(runner: Runner, user_id: str, session_id: str, query: str):
    """Streams agent response tokens as they arrive for lower perceived latency."""
    msg = Content(parts=[Part(text=query)])
    full_response = []

    for event in runner.run(user_id=user_id, session_id=session_id, new_message=msg):
        if event.content and event.content.parts:
            for part in event.content.parts:
                if hasattr(part, "text") and part.text:
                    print(part.text, end="", flush=True)
                    full_response.append(part.text)

    print()  # Newline after streaming
    return "".join(full_response)
```

## Edge Cases

- **Cache poisoning** (incorrect result cached): Implement cache busting via
  a `cache_version` key in state. Increment version when the tool logic changes.
- **`ParallelAgent` with state dependencies**: Sub-agents in `ParallelAgent`
  share the same session state. Avoid writing to the same state keys in parallel
  sub-agents — last write wins, causing race conditions.
- **Prompt compression removes critical context**: Test compressed prompts
  against the full eval suite before deploying. Never compress safety-critical
  instructions.
- **Streaming not supported by LLM response type**: Some tool-heavy responses
  don't stream text. Only attempt streaming for final text responses.

## Integration Notes

- **`google-adk-performance-benchmarking`**: Identifies the bottleneck that
  this skill addresses. Always benchmark before and after optimization.
- **`google-adk-tracing`**: Provides span-level data identifying which specific
  call is slowest. Use trace data to confirm the optimization target.
- **`google-adk-cost-tracking`**: Caching and model downgrades also reduce cost.
  Measure cost impact alongside latency improvement.
- **`google-adk-automated-evaluation`**: Run the full eval suite after every
  optimization to ensure quality is not degraded.
