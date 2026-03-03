---
name: throughput-optimization
description: >
  Use this skill when maximizing the number of Google ADK agent requests
  processed per unit time (requests/second) in a production deployment. Covers
  async request handling, connection pooling, horizontal scaling, request batching,
  rate limit management, concurrency configuration, and performance monitoring
  for high-throughput ADK deployments.
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

This skill maximizes the request throughput of Google ADK agent deployments —
the number of user requests processed per second. Throughput constraints in ADK
deployments arise from: synchronous request handling blocking the event loop,
Gemini API quota limits (requests per minute, tokens per minute), blocking I/O
in tool functions, insufficient horizontal scaling, and inefficient connection
management. This skill applies ADK-native and infrastructure-level strategies to
maximize requests-per-second while maintaining per-request quality and latency SLOs.

## When to Use

- When the ADK service cannot keep up with incoming request volume (queue backlog growing).
- When load testing reveals that throughput plateaus below target QPS.
- When Gemini API rate limiting (429 errors) is limiting throughput.
- When synchronous tool calls are blocking the async event loop.
- When a single-instance deployment needs to scale to handle burst traffic.

## When NOT to Use

- Do not optimize throughput at the expense of per-request quality or latency SLO.
- Do not use `asyncio.gather()` for tool calls with data dependencies.
- Do not scale horizontally without confirming that the session backend supports
  concurrent multi-instance access.

## Google ADK Context

- **`Runner.run()` is a generator**: The ADK runner is synchronous by default.
  For high throughput, wrap in `asyncio` and use thread pool executors for
  concurrent request handling.
- **Gemini API quotas**: `gemini-2.0-flash` default: 1000 RPM, 4M TPM. Quota
  increases available via GCP console. Design for quota-aware backpressure.
- **Async ADK**: `Runner.run_async()` (if available) or wrap `runner.run()` in
  `asyncio.to_thread()` for non-blocking request handling.
- **Connection pooling**: Use a persistent `httpx.AsyncClient` with connection
  pooling for all tool HTTP calls. Avoid creating a new connection per request.
- **Session service throughput**: `InMemorySessionService` handles unlimited
  concurrent reads/writes (in-process). `DatabaseSessionService` is limited by
  database connection pool size.
- **Rate limiter pattern**: Track rolling request count per minute. When approaching
  the quota, queue excess requests with exponential backoff retry.
- **Horizontal scaling**: Multiple ADK service replicas behind a load balancer.
  Requires a shared session backend (`DatabaseSessionService` or `VertexAiSessionService`).

## Capabilities

- Handles concurrent ADK requests asynchronously via thread pool.
- Implements Gemini API quota-aware rate limiting with backpressure.
- Uses persistent HTTP connection pooling in tool functions.
- Scales horizontally with shared persistent session backends.
- Batches independent requests for shared downstream API calls.
- Monitors requests-per-second and quota utilization in real time.

## Step-by-Step Instructions

1. **Measure baseline throughput** under load:
   ```python
   import asyncio, time
   async def load_test(runner, n_concurrent=10, n_total=100):
       sem = asyncio.Semaphore(n_concurrent)
       async def one():
           async with sem:
               await asyncio.to_thread(run_one_request, runner)
       start = time.perf_counter()
       await asyncio.gather(*[one() for _ in range(n_total)])
       rps = n_total / (time.perf_counter() - start)
       return rps
   ```

2. **Enable async request handling** via thread pool:
   ```python
   from concurrent.futures import ThreadPoolExecutor
   executor = ThreadPoolExecutor(max_workers=50)

   @app.post("/query")
   async def handle_query(request: Request):
       loop = asyncio.get_event_loop()
       response = await loop.run_in_executor(executor, run_agent, request)
       return response
   ```

3. **Use connection pooling** in tool HTTP calls:
   ```python
   import httpx
   HTTP_CLIENT = httpx.AsyncClient(limits=httpx.Limits(max_connections=100, max_keepalive_connections=20))

   async def search_flights_async(origin: str, destination: str) -> dict:
       resp = await HTTP_CLIENT.get(f"https://api.flightsearch.com/", params={"from": origin, "to": destination})
       return resp.json()
   ```

4. **Implement API quota rate limiter**:
   ```python
   import asyncio
   class RateLimiter:
       def __init__(self, rpm_limit: int):
           self.rpm_limit = rpm_limit
           self.requests = []
       async def acquire(self):
           now = asyncio.get_event_loop().time()
           self.requests = [t for t in self.requests if now - t < 60]
           if len(self.requests) >= self.rpm_limit:
               await asyncio.sleep(61 - (now - self.requests[0]))
           self.requests.append(now)
   ```

5. **Configure `DatabaseSessionService` connection pool**:
   ```python
   db_url = "postgresql+asyncpg://user:pass@host/db?pool_size=20&max_overflow=40"
   session_service = DatabaseSessionService(db_url=db_url)
   ```

6. **Monitor throughput** with Prometheus or Cloud Monitoring.

## Input Format

```python
{
  "operation": "concurrent_request_handling",
  "execution_time_ms": 2800,
  "token_usage": 4200,
  "memory_usage_mb": 256,
  "cost_estimate": 0.0021,
  "retrieval_latency_ms": 180,
  "cache_available": True,
  "parallelizable": True,
  "context_size_tokens": 4200
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "async_handling + connection_pooling + rate_limiter",
  "performance_improvement": {
    "latency_reduction_ms": 0,
    "token_reduction": 0,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.0,
    "throughput_improvement": 4.5
  }
}
```

## Error Handling

- **429 Quota Exceeded** (Gemini API rate limit): Implement exponential backoff
  retry with jitter: `wait = base * 2^attempt + random.uniform(0, 0.5)`.
- **Thread pool exhausted** (`ThreadPoolExecutor` full): Increase `max_workers`.
  Profile thread pool utilization. Move blocking I/O to async.
- **Database connection pool exhausted** (all connections in use): Increase
  `pool_size`. Optimize query performance. Add read replicas.
- **Memory exhaustion under high concurrency**: Reduce per-request memory via
  compression and state trimming. Add per-request OOM protection.
- **Throughput plateau despite scaling** (bottleneck not the service): Profile
  downstream dependencies. Identify the real bottleneck (DB, Gemini API, tool APIs).

## Examples

### Example 1 — High-throughput async ADK server (FastAPI)

```python
import asyncio
from typing import AsyncGenerator
from concurrent.futures import ThreadPoolExecutor
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from google.adk.runners import Runner
from google.adk.sessions import DatabaseSessionService
from google.genai.types import Content, Part
import os

MAX_WORKERS = int(os.environ.get("THREAD_POOL_WORKERS", 50))
executor = ThreadPoolExecutor(max_workers=MAX_WORKERS)
session_service = DatabaseSessionService(
    db_url=os.environ["DATABASE_URL"] + "?pool_size=20&max_overflow=40"
)

@asynccontextmanager
async def lifespan(app_: FastAPI):
    yield
    executor.shutdown(wait=False)

app = FastAPI(lifespan=lifespan)

def _run_agent_sync(runner: Runner, user_id: str, session_id: str, query: str) -> str:
    msg = Content(parts=[Part(text=query)])
    for event in runner.run(user_id=user_id, session_id=session_id, new_message=msg):
        if event.is_final_response() and event.content and event.content.parts:
            return event.content.parts[0].text or ""
    return ""

@app.post("/query")
async def query_endpoint(user_id: str, session_id: str, query: str, runner: Runner = None):
    loop = asyncio.get_event_loop()
    response = await loop.run_in_executor(executor, _run_agent_sync, runner, user_id, session_id, query)
    return {"response": response}
```

## Edge Cases

- **Hot session** (one session receiving very high QPS): Sessions are serialized
  on writes. Route one session to one replica instance to avoid write contention.
- **Bursty traffic** (10x normal QPS for 30 seconds): Implement request queuing.
  Reject requests when queue depth > MAX_QUEUE_DEPTH with a `503 Service Unavailable`.
- **Cost spike under high throughput**: Throughput optimization can amplify cost.
  Set per-minute token budgets in alerting. Cap throughput below quota limits.

## Integration Notes

- **`google-adk-parallelization-strategies`**: Parallelism within a request.
  Throughput optimization is about parallelism across requests.
- **`google-adk-scalability-optimization`**: Horizontal scaling is the infrastructure
  layer of throughput optimization.
- **`google-adk-latency-optimization`**: Lower per-request latency reduces total
  processing time, indirectly increasing throughput.
- **`google-adk-caching-strategies`**: Cache hits require no Gemini API call,
  dramatically increasing the effective throughput.
