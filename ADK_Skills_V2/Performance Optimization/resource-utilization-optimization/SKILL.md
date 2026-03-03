---
name: resource-utilization-optimization
description: >
  Use this skill when optimizing the use of compute, memory, network, and API
  quota resources in a Google ADK agent deployment to minimize waste, prevent
  resource exhaustion, and maximize resource efficiency under both normal and
  peak load conditions. Covers CPU/memory profiling, connection pool sizing,
  Gemini API quota utilization, async I/O, and resource monitoring.
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

This skill optimizes the efficient utilization of all resources in a Google ADK
deployment: compute (CPU, event loop), memory (heap, session storage), network
(HTTP connections, bandwidth), and API quotas (Gemini RPM/TPM, external tool APIs).
Poor resource utilization manifests as: CPU-bound tool processing blocking the
async loop, memory leaks consuming heap, excessive network connections per request,
Gemini API quota exhausted during peak load, and underutilized horizontal replicas.
This skill provides profiling, diagnosis, and optimization strategies for each
resource dimension.

## When to Use

- When CPU utilization is consistently > 80% under normal load.
- When memory usage grows over time (memory leak pattern).
- When network connections are not being reused (high connection setup overhead).
- When Gemini API quota is frequently exhausted during peak hours.
- When compute resources are allocated but underutilized (overprovisioned).

## When NOT to Use

- Do not optimize resource utilization before understanding the actual bottleneck
  — always profile first.
- Do not sacrifice code readability for micro-optimizations — measure the impact
  first.
- Do not reduce instance count without load testing the reduced configuration first.

## Google ADK Context

- **CPU utilization**: ADK tool functions should be non-blocking async. CPU-bound
  processing (data parsing, compression) should run in `asyncio.to_thread()`.
- **Memory utilization**: Per-session state in `InMemorySessionService` consumes
  RAM. Switch to `DatabaseSessionService` for memory offload. Large tool results
  should be stored compressed or truncated.
- **Network connections**: Each HTTP tool call should use persistent connections
  via `httpx.AsyncClient` connection pool. Avoid `requests.get()` (blocking, no pooling).
- **Gemini API quota**: Monitor `prompt_token_count` and request count per minute.
  Implement quota-aware rate limiting. Use caching to reduce API calls.
- **Event loop**: Blocking operations in async tool functions starve the event loop.
  Use `asyncio.to_thread()` for any blocking I/O or CPU-intensive computation.
- **Horizontal scaling**: ADK service replicas behind a load balancer. Requires
  shared session backend. Monitor per-replica CPU and memory.

## Capabilities

- Profiles CPU, memory, and network utilization in ADK deployments.
- Moves blocking tool operations to thread pools for event loop preservation.
- Implements HTTP connection pooling for network efficiency.
- Monitors Gemini API quota usage and implements backpressure.
- Tracks and evicts leaked session state for memory efficiency.
- Configures horizontal scaling with shared session backend.

## Step-by-Step Instructions

1. **Profile CPU usage** per component:
   ```python
   import cProfile, pstats
   with cProfile.Profile() as pr:
       run_agent(query)
   stats = pstats.Stats(pr).sort_stats("cumulative")
   stats.print_stats(20)
   ```

2. **Move blocking operations to thread pool**:
   ```python
   import asyncio
   async def async_tool(data: str, tool_context) -> dict:
       result = await asyncio.to_thread(cpu_intensive_function, data)
       return result
   ```

3. **Implement shared HTTP connection pool**:
   ```python
   import httpx
   _http_client = httpx.AsyncClient(
       limits=httpx.Limits(max_connections=100, max_keepalive_connections=25),
       timeout=httpx.Timeout(10.0),
   )
   ```

4. **Track Gemini quota utilization**:
   ```python
   tool_context.state["app:total_tokens_this_minute"] = (
       int(tool_context.state.get("app:total_tokens_this_minute", 0)) + current_tokens
   )
   ```

5. **Set up memory monitoring** with alerting:
   ```python
   import psutil
   def check_memory():
       mem = psutil.virtual_memory()
       if mem.percent > 85:
           # Alert and trigger session eviction
           run_eviction_cycle(session_service)
   ```

6. **Configure resource limits** in production containers:
   ```yaml
   resources:
     requests: { cpu: "500m", memory: "1Gi" }
     limits:   { cpu: "2000m", memory: "4Gi" }
   ```

## Input Format

```python
{
  "operation": "resource_audit",
  "execution_time_ms": 3200,
  "token_usage": 8500,
  "memory_usage_mb": 480,
  "cost_estimate": 0.0042,
  "retrieval_latency_ms": 220,
  "cache_available": True,
  "parallelizable": True,
  "context_size_tokens": 8500
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "async_io + connection_pooling + memory_eviction",
  "performance_improvement": {
    "latency_reduction_ms": 400,
    "token_reduction": 0,
    "memory_reduction_mb": 220,
    "cost_reduction": 0.0,
    "throughput_improvement": 2.1
  }
}
```

## Error Handling

- **Event loop blocked** (latency spike under load): Use `loop.slow_callback_duration`
  to identify slow callbacks. Move them to `asyncio.to_thread()`.
- **Memory leak** (RSS grows unboundedly): Use `tracemalloc` to find the source.
  Common cause: global dict holding session references. Use `weakref.WeakValueDictionary`.
- **Connection pool exhausted** (all connections in use): Increase pool size.
  Reduce per-request connection hold time. Use timeouts on all HTTP calls.
- **Gemini quota exhausted** (429 errors under load): Reduce request rate via rate
  limiter. Enable aggressive caching for repeat queries. Request quota increase.

## Examples

### Example 1 — Resource-efficient tool template

```python
import asyncio
import httpx
import psutil
import logging
from google.adk.tools import ToolContext

logger = logging.getLogger("resource_opt")

# Shared async HTTP client (module-level singleton)
_http = httpx.AsyncClient(
    limits=httpx.Limits(max_connections=50, max_keepalive_connections=10),
    timeout=httpx.Timeout(connect=5.0, read=10.0, write=5.0, pool=3.0),
)

async def fetch_data_efficiently(endpoint: str, params: dict, tool_context: ToolContext) -> dict:
    """Fetches external data using the shared async HTTP client.

    Args:
        endpoint (str): API endpoint URL.
        params (dict): Query parameters.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: API response data.
    """
    try:
        response = await _http.get(endpoint, params=params)
        response.raise_for_status()
        return response.json()
    except httpx.TimeoutException:
        logger.error(f"Timeout calling {endpoint}")
        return {"error": "Service timeout", "data": []}
    except httpx.HTTPStatusError as e:
        logger.error(f"HTTP error {e.response.status_code} calling {endpoint}")
        return {"error": f"HTTP {e.response.status_code}", "data": []}


def log_resource_snapshot(label: str = ""):
    """Logs current process resource utilization."""
    proc = psutil.Process()
    mem = proc.memory_info()
    cpu = proc.cpu_percent(interval=0.1)
    logger.info(f"[{label}] CPU: {cpu:.1f}%, RSS: {mem.rss / 1024**2:.1f}MB, VMS: {mem.vms / 1024**2:.1f}MB")
```

## Edge Cases

- **CPU spike during cold start** (model loading, DB connection initialization):
  Pre-warm the connection pool and execute a test query on startup before serving traffic.
- **Memory spike during peak traffic** (many concurrent sessions): Set
  `MAX_CONCURRENT_SESSIONS` and queue excess. Return `503` when the queue is full.
- **Network bandwidth saturation** (large tool response payloads): Compress API
  responses. Use `Content-Encoding: gzip` on all tool API calls.

## Integration Notes

- **`google-adk-memory-usage-optimization`**: Covers ADK-specific memory strategies.
  This skill covers the infrastructure-level resource picture.
- **`google-adk-throughput-optimization`**: Throughput optimization requires
  efficient resource utilization as a prerequisite.
- **`google-adk-scalability-optimization`**: Resource efficiency is the foundation
  for horizontal scalability.
- **`google-adk-caching-strategies`**: Every cache hit reduces CPU, memory, network,
  and API quota usage simultaneously.
