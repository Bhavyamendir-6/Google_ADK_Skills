---
name: latency-optimization
description: >
  Use this skill when reducing end-to-end response latency in a Google ADK
  LlmAgent pipeline. Covers measuring and profiling ADK agent latency per stage
  (model inference, tool execution, retrieval), applying streaming for
  time-to-first-token reduction, selecting lower-latency model variants,
  parallelizing independent tool calls via ParallelAgent, and caching
  frequently repeated computations.
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

This skill reduces end-to-end latency for Google ADK agent pipelines. Latency
is the elapsed time from receiving a user message to delivering the first
(streaming) or complete (non-streaming) response. In ADK agents, latency is
composed of: model inference time, multi-turn reasoning iterations, sequential
tool execution time, retrieval latency, and network round-trips. This skill
identifies the dominant latency source and applies targeted ADK-native optimizations:
streaming, model variant selection, parallel tool execution, caching, and context
window reduction.

## When to Use

- When P95 or P99 response latency exceeds the acceptable SLO (e.g., > 5s for chat).
- When profiling reveals model inference or tool execution as the dominant latency source.
- When non-streaming agents can be switched to streaming to reduce perceived latency.
- When sequential tool calls can be parallelized.
- When expensive tool results are recomputed on every turn (caching opportunity).
- When the context window is large and reducing it would speed up inference.

## When NOT to Use

- Do not apply latency optimizations that trade accuracy for speed without measuring quality impact.
- Do not switch to a smaller model blindly — validate output quality first.
- Do not parallelize tool calls that have data dependencies (output of A → input of B).

## Google ADK Context

- **Model inference latency**: The dominant factor. Reduced by: smaller model,
  lower `max_output_tokens`, reduced context window, streaming for TTFT.
- **`Runner.run()` generator**: Yields streaming events. Consuming the first
  text event gives time-to-first-token.
- **`ParallelAgent`**: Executes sub-agents concurrently. Reduces latency for
  independent data-gathering steps.
- **`gemini-2.0-flash`**: Lower latency than `gemini-2.0-pro`. Use Flash for
  latency-sensitive paths.
- **`gemini-2.0-flash-lite`**: Lowest latency Gemini model. Use for classification
  and routing where max capability is not required.
- **`tool_context.state` caching**: Cache expensive tool results with TTL to
  avoid redundant slow calls.
- **`max_output_tokens`**: Reducing this cuts inference time proportionally.
- **ADK Agent Execution Lifecycle**: Multi-turn reasoning (agent calls model,
  model calls tool, agent calls model again) multiplies latency. Reduce
  reasoning iterations with better instructions.

## Capabilities

- Measures per-stage latency from ADK event timing.
- Reduces time-to-first-token via streaming response delivery.
- Parallelizes independent tool calls using `ParallelAgent`.
- Reduces inference latency by selecting `gemini-2.0-flash` or `flash-lite`.
- Reduces context window size to cut inference time.
- Caches repeated tool results with TTL in `tool_context.state`.
- Monitors latency improvements post-optimization.

## Step-by-Step Instructions

1. **Measure baseline latency** end-to-end and per stage:
   ```python
   import time
   start = time.perf_counter()
   for event in runner.run(...):
       if event.is_final_response():
           e2e_ms = (time.perf_counter() - start) * 1000
   ```

2. **Identify the dominant latency source**:
   - Model inference: time between tool call event and next tool/final response event.
   - Tool execution: time inside tool function bodies.
   - Retrieval: time for vector store queries.

3. **Enable streaming** for TTFT reduction (no code change needed — `Runner.run()`
   is always a generator; ensure frontend consumes partial events).

4. **Switch to Flash/Flash-Lite** for latency-sensitive sub-agents:
   ```python
   agent = LlmAgent(name="fast_agent", model="gemini-2.0-flash-lite", ...)
   ```

5. **Parallelize independent data gathering** with `ParallelAgent`:
   ```python
   from google.adk.agents import ParallelAgent
   gather = ParallelAgent(name="gather", sub_agents=[weather_agent, price_agent])
   ```

6. **Reduce `max_output_tokens`** when responses are consistently short:
   ```python
   config = GenerateContentConfig(max_output_tokens=256)
   ```

7. **Cache slow tool results** with TTL:
   ```python
   cache_key = f"cache:flight_search:{hash(query)}"
   if not tool_context.state.get(cache_key):
       result = slow_search(query)
       tool_context.state[cache_key] = result
       tool_context.state[f"{cache_key}_expires_at"] = time.time() + 300
   ```

8. **Reduce context window** by injecting only relevant state and removing unused tools.

9. **Validate quality** after optimization using `response_match_score` eval.

## Input Format

```python
{
  "operation": "flight_search",
  "execution_time_ms": 4200,
  "token_usage": 3500,
  "memory_usage_mb": 128,
  "cost_estimate": 0.0021,
  "retrieval_latency_ms": 350,
  "cache_available": True,
  "parallelizable": True,
  "context_size_tokens": 12000
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "parallel_tools + flash_model + caching",
  "performance_improvement": {
    "latency_reduction_ms": 1800,
    "token_reduction": 0,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.0,
    "throughput_improvement": 1.75
  }
}
```

## Error Handling

- **Streaming not improving UX**: Check whether the frontend buffers the response.
  Set `X-Accel-Buffering: no` and verify the SSE consumer reads partial events.
- **ParallelAgent causes race conditions** (sub-agents write to the same state key):
  Use unique `output_key` per sub-agent in `ParallelAgent`.
- **Flash-Lite output quality insufficient**: Revert to Flash. Use a `SequentialAgent`:
  Flash-Lite for classification → Flash for generation.
- **Cache TTL too long** (stale results): Shorten TTL. Add a cache invalidation
  tool that deletes specific cache keys.
- **Latency regression after optimization**: Run before/after ADK eval comparison.
  Revert the last change and re-profile.

## Examples

### Example 1 — Latency-optimized parallel flight pipeline

```python
import time
from google.adk.agents import LlmAgent, ParallelAgent, SequentialAgent
from google.genai.types import GenerateContentConfig
from google.adk.tools import ToolContext

# Flash-Lite config for fast sub-agents
FAST_CONFIG = GenerateContentConfig(temperature=0.1, max_output_tokens=512)

# Parallel data gathering (independent queries)
weather_agent = LlmAgent(name="weather", model="gemini-2.0-flash-lite",
                          instruction="Get weather for {destination?}", tools=[get_weather],
                          generate_content_config=FAST_CONFIG, output_key="weather_data")

price_agent = LlmAgent(name="price", model="gemini-2.0-flash-lite",
                        instruction="Get flight prices for {route?}", tools=[get_prices],
                        generate_content_config=FAST_CONFIG, output_key="price_data")

gather = ParallelAgent(name="data_gather", sub_agents=[weather_agent, price_agent])

# Final response with Flash (higher quality, but receives pre-gathered data)
responder = LlmAgent(name="responder", model="gemini-2.0-flash",
                      instruction="Weather: {weather_data?}\nPrices: {price_data?}\nHelp the user.",
                      generate_content_config=GenerateContentConfig(max_output_tokens=400))

pipeline = SequentialAgent(name="flight_pipeline", sub_agents=[gather, responder])
```

### Example 2 — Per-stage latency measurement

```python
import time
from google.adk.runners import Runner
from google.genai.types import Content, Part

def measure_latency_stages(runner: Runner, user_id: str, session_id: str, query: str) -> dict:
    """Measures time-to-first-token and total end-to-end latency."""
    msg = Content(parts=[Part(text=query)])
    start = time.perf_counter()
    ttft = None
    total_ms = 0

    for event in runner.run(user_id=user_id, session_id=session_id, new_message=msg):
        if ttft is None and event.content and event.content.parts:
            for part in event.content.parts:
                if hasattr(part, "text") and part.text:
                    ttft = round((time.perf_counter() - start) * 1000, 2)
                    break
        if event.is_final_response():
            total_ms = round((time.perf_counter() - start) * 1000, 2)

    return {"ttft_ms": ttft, "total_latency_ms": total_ms}
```

## Edge Cases

- **Parallel agent with one very slow sub-agent**: `ParallelAgent` waits for all
  sub-agents to complete. Identify the slow agent and apply caching or model downgrade.
- **Very large context window** slowing inference: Use context summarization to
  reduce history to < 5000 tokens before injection.
- **Tool latency > model inference latency**: Profile each tool function. Optimize
  slow tools with async HTTP calls and connection pooling.

## Integration Notes

- **`google-adk-streaming-responses`**: Streaming is the primary TTFT optimization.
- **`google-adk-caching-strategies`**: Tool result caching eliminates repeated
  slow calls.
- **`google-adk-parallelization-strategies`**: Covers `ParallelAgent` in depth.
- **`google-adk-model-selection-optimization`**: Model downgrade is a key
  inference latency lever.
- **`google-adk-context-optimization`**: Reducing context window size reduces
  inference time proportionally.
