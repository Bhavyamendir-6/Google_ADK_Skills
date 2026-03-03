---
name: parallelization-strategies
description: >
  Use this skill when implementing parallel execution of independent agent
  operations in a Google ADK pipeline to reduce total wall-clock time. Covers
  using ParallelAgent for concurrent sub-agent execution, identifying
  data-independent operations that can run simultaneously, managing state
  namespacing to prevent parallel write conflicts, and measuring parallelization
  speedup in production ADK deployments.
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

This skill implements parallel execution strategies for Google ADK agent pipelines
to reduce total wall-clock time when multiple operations are independent. ADK's
`ParallelAgent` executes multiple sub-agents concurrently in separate async task
groups, reducing the time for "gather data from N sources" patterns from O(N×t)
to O(max(t)). This skill covers identifying parallelizable operations, designing
state namespacing to prevent write conflicts, choosing between `ParallelAgent`
and Python `asyncio.gather()` for tool-level parallelism, and validating
correctness with sequential equivalency tests.

## When to Use

- When a pipeline has multiple independent data-gathering steps that currently
  run serially (e.g., fetch weather + fetch flight prices + fetch hotel rates).
- When latency profiling shows multiple tool calls in sequence where each is
  independent of the previous.
- When the execution graph has a "fan-out → fan-in" pattern.
- When each data source is slow (> 500ms) and parallelizing would meaningfully
  reduce total time.

## When NOT to Use

- Do not parallelize operations with data dependencies (step B needs output of step A).
- Do not use `ParallelAgent` when sub-agents must write to the same `output_key`
  — last write wins, causing non-deterministic results.
- Do not parallelize when the overhead of spawning sub-agents exceeds the potential
  gain (operations taking < 100ms each).

## Google ADK Context

- **`ParallelAgent`**: `from google.adk.agents import ParallelAgent`. Executes
  all `sub_agents` concurrently via asyncio tasks. The pipeline waits for all
  to complete before proceeding.
- **`output_key`**: Each sub-agent in a `ParallelAgent` must use a unique
  `output_key` to prevent concurrent state write conflicts.
- **`SequentialAgent` + `ParallelAgent`**: Use `SequentialAgent` as the outer
  wrapper: pre-processing → `ParallelAgent` (fan-out) → post-processing (fan-in).
- **`asyncio.gather()` in tools**: For tool-level parallelism within a single
  agent's tool execution, use `asyncio.gather()` for independent async operations.
- **`before_agent_callback` in `ParallelAgent`**: Callbacks run per sub-agent.
  Use them for sub-agent-specific pre-processing.
- **Speedup estimation**: `sequential_time = sum(t_i)`. `parallel_time = max(t_i)`.
  `speedup = sequential_time / parallel_time`.

## Capabilities

- Executes independent sub-agents concurrently via `ParallelAgent`.
- Designs unique `output_key` namespacing for parallel sub-agents.
- Patterns fan-out → fan-in in `SequentialAgent` + `ParallelAgent` pipelines.
- Uses `asyncio.gather()` for parallel HTTP calls within tool functions.
- Measures parallelization speedup (theoretical and observed).
- Validates parallel output equivalency against sequential baseline.

## Step-by-Step Instructions

1. **Identify independent operations** in the current sequential pipeline:
   - Draw the execution DAG. Nodes with no dependency edges can be parallelized.

2. **Create sub-agents** for each independent operation with unique `output_key`:
   ```python
   weather_agent = LlmAgent(..., output_key="weather_data")
   price_agent = LlmAgent(..., output_key="price_data")
   hotel_agent = LlmAgent(..., output_key="hotel_data")
   ```

3. **Wrap in `ParallelAgent`**:
   ```python
   from google.adk.agents import ParallelAgent
   data_gather = ParallelAgent(
       name="parallel_gather",
       sub_agents=[weather_agent, price_agent, hotel_agent],
   )
   ```

4. **Compose with `SequentialAgent`** for the full pipeline:
   ```python
   pipeline = SequentialAgent(
       name="trip_planner",
       sub_agents=[query_parser, data_gather, trip_summarizer],
   )
   ```

5. **Pass parallel outputs to the next sequential agent** via state injection:
   ```python
   trip_summarizer = LlmAgent(
       instruction="Weather: {weather_data?}\nFlights: {price_data?}\nHotels: {hotel_data?}\nCreate the itinerary."
   )
   ```

6. **Measure speedup** by timing both sequential and parallel versions:
   ```python
   # theoretical speedup
   speedup = sum(t_i) / max(t_i)
   ```

7. **Validate correctness** by running both versions on the same inputs and comparing outputs.

## Input Format

```python
{
  "operation": "multi_source_data_gather",
  "execution_time_ms": 5200,
  "token_usage": 4800,
  "memory_usage_mb": 128,
  "cost_estimate": 0.0036,
  "retrieval_latency_ms": 450,
  "cache_available": False,
  "parallelizable": True,
  "context_size_tokens": 4800
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "parallel_agent",
  "performance_improvement": {
    "latency_reduction_ms": 3200,
    "token_reduction": 0,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.0,
    "throughput_improvement": 2.8
  }
}
```

## Error Handling

- **Parallel sub-agent fails** (one of N sub-agents raises an exception):
  `ParallelAgent` propagates the exception. Wrap in try/except at the runner level.
  Consider making non-critical sub-agents fault-tolerant by catching exceptions internally.
- **State write collision** (two sub-agents use the same `output_key`): Always
  verify each sub-agent has a unique `output_key`. Test with assertion.
- **Parallel speedup less than expected** (one sub-agent is much slower than others):
  Profile each sub-agent individually. Apply independent optimization to the slow one.
- **`asyncio.gather()` in non-async tool** (`RuntimeError: no running event loop`):
  Use `asyncio.run()` or ensure the tool is defined as `async def`.

## Examples

### Example 1 — Complete fan-out/fan-in trip planning pipeline

```python
from google.adk.agents import LlmAgent, ParallelAgent, SequentialAgent
from google.genai.types import GenerateContentConfig

FAST_CONFIG = GenerateContentConfig(temperature=0.1, max_output_tokens=600)

# --- Step 1: Parse the user's trip request ---
query_parser = LlmAgent(
    name="query_parser",
    model="gemini-2.0-flash-lite",
    instruction="Extract: destination, dates, budget from the user request. Store in state.",
    tools=[parse_trip_request],
    output_key="parsed_query",
)

# --- Step 2: Parallel data gathering (independent) ---
weather_agent = LlmAgent(
    name="weather_agent", model="gemini-2.0-flash-lite",
    instruction="Get weather for {destination?} on {dates?}.",
    tools=[get_weather], generate_content_config=FAST_CONFIG, output_key="weather_data"
)

flight_agent = LlmAgent(
    name="flight_agent", model="gemini-2.0-flash-lite",
    instruction="Get flight options for {destination?} on {dates?} within budget {budget?}.",
    tools=[search_flights], generate_content_config=FAST_CONFIG, output_key="flight_data"
)

hotel_agent = LlmAgent(
    name="hotel_agent", model="gemini-2.0-flash-lite",
    instruction="Get hotel options for {destination?} on {dates?} within budget {budget?}.",
    tools=[search_hotels], generate_content_config=FAST_CONFIG, output_key="hotel_data"
)

parallel_gather = ParallelAgent(
    name="parallel_data_gather",
    sub_agents=[weather_agent, flight_agent, hotel_agent],
)

# --- Step 3: Summarize (sequential, depends on all parallel results) ---
trip_planner = LlmAgent(
    name="trip_planner_summarizer",
    model="gemini-2.0-flash",
    instruction="""
Create a complete trip plan using the gathered data:
Weather: {weather_data?}
Flights: {flight_data?}
Hotels: {hotel_data?}
Budget: {budget?}
Present the best options clearly.
""",
    generate_content_config=GenerateContentConfig(max_output_tokens=800),
)

# --- Full pipeline ---
trip_pipeline = SequentialAgent(
    name="trip_planning_pipeline",
    sub_agents=[query_parser, parallel_gather, trip_planner],
)
```

### Example 2 — Parallel async HTTP calls within a single tool

```python
import asyncio
import aiohttp
from google.adk.tools import ToolContext

async def fetch_multi_source_prices(routes: list[str], tool_context: ToolContext) -> dict:
    """Fetches prices from multiple airline APIs in parallel.

    Args:
        routes (list[str]): List of route strings (e.g., 'JFK-LHR', 'JFK-CDG').
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'prices' (dict mapping route to price list).
    """
    async def fetch_one(session: aiohttp.ClientSession, route: str) -> tuple[str, list]:
        async with session.get(f"https://api.example.com/prices/{route}") as resp:
            if resp.status == 200:
                return route, (await resp.json()).get("prices", [])
            return route, []

    async with aiohttp.ClientSession() as session:
        results = await asyncio.gather(*(fetch_one(session, r) for r in routes))

    prices = {route: price_list for route, price_list in results}
    tool_context.state["multi_route_prices"] = prices
    return {"prices": prices, "routes_fetched": len(prices)}
```

## Edge Cases

- **One parallel sub-agent is much slower** (stragglers): `ParallelAgent` blocks
  until all complete. Profile individual sub-agents. Apply caching or model
  downgrade to slow stragglers.
- **Memory spike** when all sub-agents read large data simultaneously:
  Limit each sub-agent's data retrieval to essential fields only.
- **Ordering of results** from `asyncio.gather()`: Results are in input order
  regardless of completion order. No sorting needed.

## Integration Notes

- **`google-adk-latency-optimization`**: `ParallelAgent` is the highest-impact
  latency optimization for multi-source pipelines.
- **`google-adk-execution-pipeline-optimization`**: Covers the full pipeline
  design, of which parallelization is one strategy.
- **`google-adk-caching-strategies`**: Combine parallel execution with caching —
  each parallel sub-agent benefits from its own cache layer.
- **`google-adk-throughput-optimization`**: Parallelism is both a latency and
  throughput optimization strategy.
