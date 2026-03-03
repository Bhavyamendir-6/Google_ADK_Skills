---
name: execution-pipeline-optimization
description: >
  Use this skill when optimizing the design and execution order of a multi-agent
  Google ADK pipeline (SequentialAgent, ParallelAgent, LoopAgent) to minimize
  total execution time, eliminate unnecessary agent invocations, reduce pipeline
  overhead, and maximize throughput. Covers pipeline profiling, step elimination,
  short-circuit patterns, batching, and pipeline topology redesign.
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

This skill optimizes the design and execution of multi-agent ADK pipelines —
specifically `SequentialAgent`, `ParallelAgent`, `LoopAgent`, and their
compositions. Pipeline inefficiencies include: unnecessary sequential steps that
could be parallelized, redundant agent calls with the same inputs, excessive
`LoopAgent` iterations, over-complex prompt chaining adding latency without
benefit, and high pipeline setup overhead. This skill profiles each pipeline stage,
identifies and eliminates bottlenecks, and redesigns the pipeline topology for
minimal wall-clock time and optimal resource utilization.

## When to Use

- When total pipeline execution time is unacceptable and profiling reveals multiple
  contributing stages.
- When a `LoopAgent` runs more iterations than necessary.
- When `SequentialAgent` stages have no data dependencies between adjacent steps.
- When a pipeline stage is always called even when its output is not needed.
- When implementing a new pipeline and evaluating its structural design.

## When NOT to Use

- Do not restructure a pipeline that workss correctly if latency is within SLO
  — pipeline changes introduce risk.
- Do not eliminate pipeline steps without validating that the agent behaves
  correctly without them.
- Do not add parallelism to steps that have data dependencies — this causes
  incorrect results.

## Google ADK Context

- **`SequentialAgent`**: Runs sub-agents in order. Each sub-agent completes before
  the next starts. Use only when data dependencies exist between steps.
- **`ParallelAgent`**: Runs all sub-agents concurrently. Use for independent data gathering.
- **`LoopAgent`**: Repeats an sub-agent until `escalate` or `max_iterations` is reached.
  Minimize `max_iterations` and add clear termination conditions.
- **`output_key`**: Each sub-agent's output is written to a state key. The next
  agent reads from it via `{key?}`. Design output_key dependencies to identify
  sequential requirements.
- **Pipeline overhead**: Each agent invocation has ADK framework overhead (~20–50ms).
  Merge trivially simple agents into a single tool function where possible.
- **Short-circuit pattern**: Skip expensive pipeline stages when prior stages
  determine they are not needed (set a skip flag in state, check in a callback).
- **`before_agent_callback` returning `Content`**: Returns a pre-determined response
  immediately, skipping the LLM call entirely. Effective for cache hits.

## Capabilities

- Profiles per-stage execution time in ADK pipelines.
- Identifies and parallelizes independent sequential steps.
- Implements short-circuit patterns to skip unnecessary stages.
- Reduces `LoopAgent` iteration counts via better termination conditions.
- Merges redundant lightweight agents into single tool functions.
- Measures and validates pipeline speedup before production deployment.

## Step-by-Step Instructions

1. **Profile the pipeline** — measure per-stage execution time:
   ```python
   import time
   stage_times = {}
   for stage_name, agent in zip(stage_names, sub_agents):
       start = time.perf_counter()
       # Run the stage
       stage_times[stage_name] = (time.perf_counter() - start) * 1000
   ```

2. **Draw the dependency graph**: For each sub-agent, identify which output_keys
   it produces and which it consumes. Nodes with no incoming dependencies from
   the previous node can be parallelized.

3. **Parallelize independent stages** (see `parallelization-strategies`):
   ```python
   # Sequential: A → B → C → D (if B and C are independent of each other)
   # Optimized: A → ParallelAgent(B, C) → D
   parallel_bc = ParallelAgent(name="parallel_bc", sub_agents=[agent_b, agent_c])
   pipeline = SequentialAgent(name="opt_pipeline", sub_agents=[agent_a, parallel_bc, agent_d])
   ```

4. **Implement short-circuit** for expensive stages:
   ```python
   def maybe_skip_retrieval(callback_context: CallbackContext):
       if callback_context.state.get("rag_context_ready"):
           return Content(parts=[Part(text="Retrieved context already loaded.")])
       return None

   retrieval_agent = LlmAgent(..., before_agent_callback=maybe_skip_retrieval)
   ```

5. **Reduce `LoopAgent` max_iterations**:
   - Add explicit stopping criteria as state flags: `state["search_complete"] = True`
   - Check stopping flag in `before_agent_callback` → return `Content` to break loop.

6. **Merge trivial agents** into tool functions:
   - A single-step agent that extracts one field → replace with a `before_agent_callback`.
   - A no-tool LLM agent that reformats text → add reformatting to the previous
     agent's instruction.

7. **Validate correctness** via integration tests after pipeline restructuring.

## Input Format

```python
{
  "operation": "pipeline_execution",
  "execution_time_ms": 12000,
  "token_usage": 28000,
  "memory_usage_mb": 256,
  "cost_estimate": 0.014,
  "retrieval_latency_ms": 400,
  "cache_available": True,
  "parallelizable": True,
  "context_size_tokens": 28000
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "parallel_stages + short_circuit + loop_iteration_reduction",
  "performance_improvement": {
    "latency_reduction_ms": 5800,
    "token_reduction": 8000,
    "memory_reduction_mb": 64,
    "cost_reduction": 0.004,
    "throughput_improvement": 2.6
  }
}
```

## Error Handling

- **Parallelized agents produce conflicting state**: Assign unique `output_key` to
  each parallel sub-agent. Never share `output_key` across sub-agents in a `ParallelAgent`.
- **Short-circuit fires incorrectly** (skips stage when it's needed): Add logging
  to the short-circuit decision. Validate with integration test coverage.
- **LoopAgent `max_iterations` exceeded**: The loop's evaluation step is not
  detecting the stopping condition. Add explicit state flags in the loop body.
- **Merged agent is too complex**: If two agents merged into one produces worse
  quality, split them back. Prefer composability over micro-optimization.

## Examples

### Example 1 — Profiled, optimized pipeline with short-circuit

```python
import time
import logging
from typing import Optional
from google.adk.agents import LlmAgent, ParallelAgent, SequentialAgent
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content, Part

logger = logging.getLogger("pipeline_opt")

# Short-circuit: skip retrieval if context already loaded this session
def skip_if_context_loaded(callback_context: CallbackContext) -> Optional[Content]:
    if callback_context.state.get("rag_context_loaded_this_session"):
        logger.debug("Skipping retrieval — context already loaded.")
        return Content(parts=[Part(text="(Retrieval skipped — context already loaded)")])
    return None

retrieval_agent = LlmAgent(
    name="retrieval_agent",
    model="gemini-2.0-flash-lite",
    instruction="Retrieve relevant policy context for: {user_query?}",
    tools=[retrieve_policy_context],
    before_agent_callback=skip_if_context_loaded,
    output_key="retrieved_context",
)

# Independent parallel data gathering
weather_agent = LlmAgent(name="weather", model="gemini-2.0-flash-lite",
                          tools=[get_weather], output_key="weather_data",
                          instruction="Get weather for {destination?}")
pricing_agent = LlmAgent(name="pricing", model="gemini-2.0-flash-lite",
                          tools=[get_prices], output_key="pricing_data",
                          instruction="Get pricing for {route?}")

parallel_gather = ParallelAgent(name="parallel_gather", sub_agents=[weather_agent, pricing_agent])

responder = LlmAgent(
    name="responder", model="gemini-2.0-flash",
    instruction="Context: {retrieved_context?}\nWeather: {weather_data?}\nPricing: {pricing_data?}\nRespond to the user."
)

optimized_pipeline = SequentialAgent(
    name="optimized_booking_pipeline",
    sub_agents=[retrieval_agent, parallel_gather, responder],
)
```

## Edge Cases

- **Pipeline topology change requires eval**: Never change pipeline topology without
  running integration tests. Parallelizing steps that have hidden data dependencies
  causes non-deterministic failures.
- **Over-parallelized pipeline** (too many parallel stages): `ParallelAgent` spawns
  goroutines per sub-agent. More than 5 concurrent sub-agents may saturate the async
  event loop. Profile under load.
- **LoopAgent exits on first iteration** (termination condition set too eagerly):
  Validate that the loop body actually modifies the state that the termination
  condition checks.

## Integration Notes

- **`google-adk-parallelization-strategies`**: The primary technique for pipeline
  stage parallelization.
- **`google-adk-latency-optimization`**: Pipeline optimization directly reduces
  end-to-end latency.
- **`google-adk-throughput-optimization`**: A more efficient pipeline increases
  the number of requests handled per second.
- **`google-adk-caching-strategies`**: Short-circuit patterns rely on caching to
  determine when stages can be skipped.
