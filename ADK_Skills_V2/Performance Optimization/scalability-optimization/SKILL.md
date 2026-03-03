---
name: scalability-optimization
description: >
  Use this skill when designing Google ADK agent deployments to scale horizontally
  across multiple instances under increasing load. Covers stateless agent design
  with shared persistent session backends, load balancer configuration, session
  affinity strategies, auto-scaling policies, quota management for multi-instance
  deployments, and capacity planning for ADK services.
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

This skill designs Google ADK agent deployments to scale horizontally to handle
growing user loads by adding more service instances without degrading per-request
quality or correctness. Key scalability requirements for ADK agents: the session
service must be shared across instances (stateless instances); Gemini API quotas
must be managed across all instances; agent configuration must be stateless (no
in-process mutable global state); and the deployment infrastructure must support
auto-scaling based on request load signals.

## When to Use

- When a single ADK service instance cannot handle peak request volume.
- When designing a new ADK deployment for a use case with unpredictable or growing load.
- When an existing deployment needs to move from single-instance to multi-instance.
- When `InMemorySessionService` blocks horizontal scaling (users hitting different
  instances lose session state).
- When implementing auto-scaling policies for a cloud-hosted ADK service.

## When NOT to Use

- Do not apply scalability optimization for services with < 100 concurrent users —
  a single well-optimized instance is sufficient.
- Do not add horizontal scaling without first ensuring the session backend supports
  concurrent multi-instance access.
- Do not scale horizontally if the bottleneck is the Gemini API quota — scaling
  instances does not increase API quota.

## Google ADK Context

- **Stateless instance requirement**: Each ADK service instance must read session
  state from a shared backend. `InMemorySessionService` is per-process and breaks
  horizontal scaling.
- **Shared session backends**: `DatabaseSessionService` (PostgreSQL with connection
  pooling) or `VertexAiSessionService` are multi-instance-safe.
- **Gemini API quota per project**: Quota is shared across all instances. A 10x
  scale-out of instances does not increase the quota.
- **Session affinity (sticky sessions)**: Optionally route a user's requests to
  the same instance (reduces DB reads). Not required for correctness with a
  shared backend.
- **Health endpoints**: Expose `/health` and `/ready` endpoints for the load
  balancer and auto-scaler.
- **Auto-scaling signals**: CPU utilization, request queue depth, or custom metric
  (e.g., pending session count) can trigger auto-scaling.
- **Agent module statelessness**: Never store per-request state in module-level
  Python globals. Use `tool_context.state` and `callback_context.state` exclusively.

## Capabilities

- Configures shared persistent session backends for multi-instance ADK deployments.
- Implements stateless agent design with no module-level mutable state.
- Configures load balancer session affinity (optional).
- Defines auto-scaling policies based on request queue depth and CPU.
- Manages Gemini API quota across a scaled-out instance fleet.
- Exposes health and readiness endpoints for the load balancer.

## Step-by-Step Instructions

1. **Verify agent statelessness** — audit for module-level mutable globals:
   ```python
   # BAD: module-level mutable state
   session_cache = {}  # This is per-instance!

   # GOOD: use tool_context.state for per-session data
   tool_context.state["cache_key"] = value
   ```

2. **Configure a shared session backend**:
   ```python
   session_service = DatabaseSessionService(
       db_url="postgresql+asyncpg://user:pass@db-primary:5432/adk?pool_size=20&max_overflow=40"
   )
   ```

3. **Add health and readiness endpoints**:
   ```python
   from fastapi import FastAPI
   app = FastAPI()

   @app.get("/health")
   async def health():
       return {"status": "ok"}

   @app.get("/ready")
   async def ready():
       # Check DB connectivity
       try:
           await validate_db(session_service)
           return {"status": "ready"}
       except Exception:
           return {"status": "not_ready"}, 503
   ```

4. **Configure container auto-scaling** (Kubernetes HPA):
   ```yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: adk-agent-hpa
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: adk-agent
     minReplicas: 2
     maxReplicas: 20
     metrics:
     - type: Resource
       resource:
         name: cpu
         target:
           type: Utilization
           averageUtilization: 70
   ```

5. **Implement Gemini quota management across instances** by storing quota usage
   in a shared Redis counter or `app:` state, not per-instance counters.

6. **Load test** the multi-instance deployment at 2×, 5×, 10× normal load to
   validate linear throughput scaling.

## Input Format

```python
{
  "operation": "horizontal_scale_out",
  "execution_time_ms": 2800,
  "token_usage": 5500,
  "memory_usage_mb": 512,
  "cost_estimate": 0.0027,
  "retrieval_latency_ms": 180,
  "cache_available": True,
  "parallelizable": True,
  "context_size_tokens": 5500
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "horizontal_scaling + shared_db_backend + stateless_agents",
  "performance_improvement": {
    "latency_reduction_ms": 0,
    "token_reduction": 0,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.0,
    "throughput_improvement": 8.0
  }
}
```

## Error Handling

- **Session not found on a different instance**: This means `InMemorySessionService`
  is still in use. Switch to `DatabaseSessionService` immediately.
- **DB connection pool exhausted under scale-out**: Each instance has its own pool.
  Total connections = `instance_count × pool_size`. Set `pool_size` conservatively
  and monitor DB `max_connections`.
- **Gemini quota exhausted faster under scale-out**: Implement a
  distributed rate limiter (Redis-based token bucket) to share the quota across instances.
- **Auto-scaler flaps** (rapid scale-up/down): Add stabilization windows:
  `scaleUp.stabilizationWindowSeconds: 60`, `scaleDown.stabilizationWindowSeconds: 300`.

## Examples

### Example 1 — Scalable, stateless ADK agent application

```python
import os
from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import DatabaseSessionService, VertexAiSessionService

def create_session_service():
    """Creates a multi-instance-safe shared session backend."""
    env = os.environ.get("DEPLOYMENT_ENV", "dev")
    if env == "gcp":
        return VertexAiSessionService(
            project=os.environ["GOOGLE_CLOUD_PROJECT"],
            location=os.environ.get("VERTEX_AI_LOCATION", "us-central1"),
        )
    else:
        db_url = os.environ.get(
            "DATABASE_URL",
            "postgresql+asyncpg://adk:secret@localhost/adk_prod?pool_size=20&max_overflow=40&pool_timeout=10"
        )
        return DatabaseSessionService(db_url=db_url)

# Stateless agent — no module-level mutable state
agent = LlmAgent(
    name="scalable_agent",
    model="gemini-2.0-flash",
    instruction="You are a helpful assistant. {session_context?}",
    tools=[search_tool, booking_tool],
)

session_service = create_session_service()
runner = Runner(agent=agent, app_name="scalable_app", session_service=session_service)
```

### Example 2 — Distributed quota manager (Redis-backed)

```python
import time
import redis
import asyncio
import logging

logger = logging.getLogger("quota_manager")

class DistributedRateLimiter:
    """Redis-backed rate limiter shared across all ADK service instances."""

    def __init__(self, redis_url: str, key: str, rpm_limit: int):
        self.client = redis.from_url(redis_url, decode_responses=True)
        self.key = f"rate_limit:{key}"
        self.rpm_limit = rpm_limit

    async def acquire(self, timeout_seconds: float = 5.0) -> bool:
        """Acquires a rate limit slot. Blocks until available or timeout."""
        deadline = time.time() + timeout_seconds
        while time.time() < deadline:
            now = int(time.time())
            window_key = f"{self.key}:{now // 60}"  # Per-minute window
            count = self.client.incr(window_key)
            self.client.expire(window_key, 120)  # TTL 2 minutes

            if count <= self.rpm_limit:
                return True
            # Backoff and retry
            await asyncio.sleep(0.1)

        logger.warning(f"Rate limit timeout for key={self.key}")
        return False
```

## Edge Cases

- **Shared `app:` state consistency**: All instances share `app:` state through the
  session backend. Last writer wins. For atomic updates (e.g., incrementing counters),
  use the database's atomic operations, not read-modify-write in Python.
- **Rolling deployment** (old and new code running simultaneously): Both versions
  must be compatible with the same session state schema. Use schema versioning.
- **Cold start latency** (new instance spinning up): Pre-warm by running a test
  query on startup before routing real traffic. Set `initialDelaySeconds` in Kubernetes
  readiness probe.

## Integration Notes

- **`google-adk-memory-storage-backends`**: Shared backends are the foundation
  of horizontal scalability.
- **`google-adk-throughput-optimization`**: Each horizontally scaled instance
  contributes to aggregate throughput.
- **`google-adk-resource-utilization-optimization`**: Efficient per-instance
  resource use maximizes the return on each additional instance.
- **`google-adk-memory-persistence`**: Cross-instance memory persistence is
  the core capability unlocked by shared session backends.
