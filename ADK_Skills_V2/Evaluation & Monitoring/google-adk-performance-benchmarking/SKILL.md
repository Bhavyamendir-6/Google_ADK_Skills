---
name: google-adk-performance-benchmarking
description: >
  Use this skill when measuring and reporting the performance of a Google ADK
  agent across tool latency, LLM call latency, end-to-end invocation time,
  and throughput. Covers designing benchmark harnesses, running load tests,
  collecting timing metrics in tool functions and callbacks, and establishing
  performance baselines and regression gates.
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

This skill designs and executes performance benchmarks for Google ADK agents.
It measures end-to-end invocation latency, per-tool execution time, LLM call
latency, throughput (invocations per second), and error rates under load. Results
establish baseline performance profiles, detect regressions after code or model
changes, and guide latency optimization efforts.

## When to Use

- When establishing a performance baseline for a new ADK agent before production.
- When measuring the impact of a model upgrade or tool refactor on latency.
- When running load tests to determine ADK agent throughput capacity.
- When detecting performance regressions in CI/CD pipelines.
- When comparing the performance of multiple agent configurations.

## When NOT to Use

- Do not run load tests against production LLM endpoints without rate-limit
  awareness — Gemini API has per-project quotas.
- Do not use this skill for functional correctness validation — use
  `google-adk-automated-evaluation` for that.
- Do not benchmark using interactive `adk web` sessions — use programmatic
  `runner.run()` calls for reproducible benchmarks.

## Google ADK Context

- **`Runner.run()`**: The programmatic entry point for ADK agent execution.
  Used in benchmark harnesses to run the agent without the web UI.
- **`session_service.create_session()`**: Creates a fresh session for each
  benchmark run to avoid state contamination.
- **`tool_context.session_id`** / **`invocation_id`**: Correlation keys for
  timing data collection.
- **Agent Execution Lifecycle timing points**:
  - Invocation start: `before_agent_callback`.
  - LLM call: Implicit in `LlmAgent.run()`. Timed via span or callback.
  - Tool call start/end: Timed in tool functions.
  - Final response: `event.is_final_response()` marks the end of the invocation.
- **`before_agent_callback` / `after_agent_callback`**: Hooks for measuring
  total invocation time.
- **OpenTelemetry histograms**: Preferred for latency distribution recording.
  Use `meter.create_histogram("adk.invocation.latency_ms")`.

## Capabilities

- Measures per-tool latency with timing wrappers in tool functions.
- Measures end-to-end invocation latency via before/after agent callbacks.
- Runs concurrent benchmark scenarios to measure throughput.
- Establishes P50/P95/P99 latency profiles from histogram data.
- Detects latency regressions against stored baseline profiles.
- Records benchmark results to JSON for trend analysis.
- Integrates with OpenTelemetry metrics for production-grade benchmarking.

## Step-by-Step Instructions

1. **Define benchmark scenarios** — a list of representative user queries that
   cover common and critical agent tasks.

2. **Instrument tool functions** with timing wrappers:
   ```python
   import time
   def my_tool(query: str, tool_context: ToolContext) -> dict:
       start = time.perf_counter()
       result = perform_operation(query)
       latency_ms = (time.perf_counter() - start) * 1000
       record_tool_latency("my_tool", latency_ms, tool_context.session_id)
       return result
   ```

3. **Instrument agent callbacks** for end-to-end timing:
   ```python
   _invocation_starts: dict = {}
   def before_agent(cb: CallbackContext) -> None:
       _invocation_starts[cb.session_id] = time.perf_counter()

   def after_agent(cb: CallbackContext) -> None:
       start = _invocation_starts.pop(cb.session_id, None)
       if start:
           latency_ms = (time.perf_counter() - start) * 1000
           record_invocation_latency(latency_ms, cb.session_id)
   ```

4. **Write the benchmark harness**:
   ```python
   async def run_benchmark(agent, session_service, queries: list[str], app_name: str):
       results = []
       for query in queries:
           session = await session_service.create_session(app_name=app_name, user_id="bench_user")
           runner = Runner(agent=agent, app_name=app_name, session_service=session_service)
           start = time.perf_counter()
           msg = Content(parts=[Part(text=query)])
           for event in runner.run(user_id="bench_user", session_id=session.id, new_message=msg):
               if event.is_final_response():
                   break
           latency_ms = (time.perf_counter() - start) * 1000
           results.append({"query": query, "latency_ms": round(latency_ms, 2)})
       return results
   ```

5. **Run concurrent load test**:
   ```python
   import asyncio
   tasks = [run_single_invocation(runner, query) for query in queries * 10]
   results = await asyncio.gather(*tasks)
   ```

6. **Compute P50/P95/P99** from results:
   ```python
   import statistics
   latencies = sorted(r["latency_ms"] for r in results)
   p50 = statistics.median(latencies)
   p95 = latencies[int(len(latencies) * 0.95)]
   p99 = latencies[int(len(latencies) * 0.99)]
   ```

7. **Store baseline** and compare on each CI run:
   ```python
   baseline = {"p50": 450, "p95": 1200, "p99": 2500}
   assert p95 <= baseline["p95"] * 1.2, f"P95 regression: {p95}ms vs baseline {baseline['p95']}ms"
   ```

## Input Format

```python
{
  "agent_id": "my_agent",
  "session_id": "bench_sess_001",
  "tool_name": "search_products",
  "execution_time_ms": 342,
  "status": "success",
  "error": None,
  "metadata": {
    "benchmark_scenario": "product_search",
    "concurrent_users": 10,
    "iterations": 100,
    "baseline_p95_ms": 1200
  }
}
```

## Output Format

```python
{
  "metric_recorded": True,
  "evaluation_result": "pass",
  "latency_ms": 342,
  "cost_estimate": 0.0,
  "issues_detected": [],
  "benchmark_results": {
    "scenario": "product_search",
    "iterations": 100,
    "p50_ms": 410,
    "p95_ms": 980,
    "p99_ms": 1450,
    "error_rate": 0.01,
    "throughput_rps": 4.2,
    "regression_detected": False
  }
}
```

## Error Handling

- **Gemini API rate limit hit during load test**: Implement exponential backoff
  for `ResourceExhausted` errors. Reduce concurrency and retry.
- **Benchmark session contamination**: Always create a fresh session per
  benchmark iteration. Never reuse sessions across benchmark runs.
- **Timing outliers** from cold starts: Warm up with 5 pre-benchmark runs and
  discard their results before recording baseline data.
- **Missing telemetry data**: If callback timing misses (session_id not found
  in dict), log a warning and exclude from P-percentile calculation.
- **Flaky P99** from network jitter: Run benchmarks from the same network zone
  as the agent deployment. Run at least 100 iterations for stable P99.

## Examples

### Example 1 — Complete benchmark harness

```python
import asyncio
import time
import json
import statistics
from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai.types import Content, Part

async def benchmark_agent(agent_module_name: str, queries: list[str], iterations: int = 50):
    """Runs a performance benchmark for an ADK agent."""
    from importlib import import_module
    module = import_module(agent_module_name)
    agent = module.root_agent

    session_service = InMemorySessionService()
    app_name = "benchmark"
    latencies = []

    for i in range(iterations):
        query = queries[i % len(queries)]
        session = await session_service.create_session(
            app_name=app_name, user_id="bench_user"
        )
        runner = Runner(agent=agent, app_name=app_name, session_service=session_service)
        msg = Content(parts=[Part(text=query)])

        start = time.perf_counter()
        for event in runner.run(user_id="bench_user", session_id=session.id, new_message=msg):
            if event.is_final_response():
                break
        latency_ms = round((time.perf_counter() - start) * 1000, 2)
        latencies.append(latency_ms)

    latencies_sorted = sorted(latencies)
    n = len(latencies_sorted)
    results = {
        "iterations": iterations,
        "p50_ms": round(statistics.median(latencies_sorted), 2),
        "p95_ms": round(latencies_sorted[int(n * 0.95)], 2),
        "p99_ms": round(latencies_sorted[int(n * 0.99)], 2),
        "mean_ms": round(statistics.mean(latencies_sorted), 2),
        "min_ms": latencies_sorted[0],
        "max_ms": latencies_sorted[-1],
    }
    print(json.dumps(results, indent=2))
    return results

if __name__ == "__main__":
    queries = ["What products do you have?", "Search for laptops", "Show me the latest deals"]
    asyncio.run(benchmark_agent("my_agent", queries, iterations=100))
```

### Example 2 — Tool latency wrapper

```python
import time
import json
import logging
from google.adk.tools import ToolContext

_tool_latencies: dict[str, list] = {}
logger = logging.getLogger("benchmark")

def timed_tool(tool_func):
    """Decorator that records tool execution latency."""
    def wrapper(*args, tool_context: ToolContext, **kwargs):
        start = time.perf_counter()
        try:
            result = tool_func(*args, tool_context=tool_context, **kwargs)
            latency_ms = round((time.perf_counter() - start) * 1000, 2)
            _tool_latencies.setdefault(tool_func.__name__, []).append(latency_ms)
            logger.info(json.dumps({"event": "tool_latency", "tool": tool_func.__name__,
                                    "latency_ms": latency_ms, "status": "success"}))
            return result
        except Exception as exc:
            latency_ms = round((time.perf_counter() - start) * 1000, 2)
            _tool_latencies.setdefault(tool_func.__name__, []).append(latency_ms)
            logger.error(json.dumps({"event": "tool_latency", "tool": tool_func.__name__,
                                     "latency_ms": latency_ms, "status": "error"}))
            raise
    return wrapper
```

### Example 3 — CI regression gate

```python
import json
import os

def check_latency_regression(results: dict, baseline_file: str = "benchmarks/baseline.json"):
    """Checks benchmark results against stored baseline. Raises on regression."""
    if not os.path.exists(baseline_file):
        with open(baseline_file, "w") as f:
            json.dump(results, f, indent=2)
        print(f"Baseline saved to {baseline_file}")
        return

    with open(baseline_file) as f:
        baseline = json.load(f)

    REGRESSION_THRESHOLD = 1.2  # 20% degradation allowed
    for metric in ["p50_ms", "p95_ms", "p99_ms"]:
        if results[metric] > baseline[metric] * REGRESSION_THRESHOLD:
            raise AssertionError(
                f"Performance regression in {metric}: "
                f"{results[metric]}ms vs baseline {baseline[metric]}ms "
                f"(>{REGRESSION_THRESHOLD:.0%} increase)"
            )
    print("No performance regression detected.")
```

## Edge Cases

- **Cold start latency**: ADK initializes session service and model on first run.
  First 2–3 invocations are cold starts. Discard them from benchmark data.
- **Gemini model variability**: LLM response time varies by token count and
  server load. Benchmark at different times of day for realistic estimates.
- **Multi-agent pipeline benchmarking**: Instrument each sub-agent separately
  to identify which agent in the pipeline contributes most to latency.
- **Tool with external I/O** (HTTP calls, DB queries): Benchmark the tool in
  isolation with mocked I/O first, then in full pipeline with real I/O to
  separate ADK overhead from external dependency latency.

## Integration Notes

- **`google-adk-tracing`**: Span data from OTel tracing is the most precise
  source of latency data. Use tracing spans as the primary latency data source.
- **`google-adk-latency-optimization`**: Benchmarks identify bottlenecks;
  `latency-optimization` skill implements the fixes.
- **`google-adk-cost-tracking`**: Benchmark runs consume Gemini API tokens.
  Track cost per benchmark run using the `cost-tracking` skill.
- **`google-adk-automated-evaluation`**: Performance benchmarks can be added
  as `pytest` tests alongside functional evaluation tests in CI.
