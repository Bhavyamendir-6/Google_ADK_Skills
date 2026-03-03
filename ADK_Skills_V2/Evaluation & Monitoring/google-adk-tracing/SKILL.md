---
name: google-adk-tracing
description: >
  Use this skill when a Google ADK agent needs distributed tracing of tool
  calls, LLM invocations, and sub-agent execution — correlating all spans
  within a single invocation using session_id and invocation_id. Covers
  OpenTelemetry span design, ADK Web UI Trace View, span propagation across
  multi-agent pipelines, and integration with Google Cloud Trace.
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

This skill implements distributed tracing for Google ADK agents. Tracing
provides a detailed, time-ordered record of every operation within an agent
invocation — LLM call, tool call, sub-agent call — enabling developers to
understand exactly where time is spent, where failures occur, and how
multi-agent pipelines execute. ADK's built-in Trace View provides basic
per-session tracing; this skill extends it to production-grade distributed
tracing via OpenTelemetry.

## When to Use

- When debugging slow agent responses and needing to identify which tool or
  LLM call is the bottleneck.
- When tracing execution across multi-agent pipelines (`SequentialAgent`,
  `ParallelAgent`) to understand sub-agent interaction.
- When integrating ADK agents with a distributed tracing system (Google Cloud
  Trace, Jaeger, Zipkin).
- When correlating ADK agent traces with backend service traces in a
  microservices architecture.
- When the ADK Web UI Trace View is insufficient for production-scale or
  multi-tenant analysis.

## When NOT to Use

- Do not create spans for every Python function — only trace at ADK execution
  lifecycle boundaries (tool call, LLM call, sub-agent call, callback).
- Do not trace in local development without a backend — use ADK Web UI Trace
  View instead. It requires no setup.
- Do not propagate traces through insecure channels without W3C Trace Context
  header validation.

## Google ADK Context

- **ADK Web UI Trace View**: Built-in, session-level trace viewer. Groups events
  by user message. Shows Event data, model Request/Response, tool call Graph.
  Available at `adk web <agent_path>`. Activated by clicking the Trace tab.
- **`invocation_id`**: Unique identifier for a single agent invocation (one user
  turn). All events and tool calls within a turn share the same `invocation_id`.
  Use as the root span ID for distributed tracing.
- **`tool_context.session_id`**: Session-level trace correlation key.
- **`tool_context.user_id`**: User-level trace attribute.
- **`event.invocation_id`**: Available on every ADK `Event` object. Used to
  group trace events.
- **Agent Execution Lifecycle trace points**:
  - `before_agent_callback` → invocation root span start.
  - Tool function entry/exit → child spans.
  - `after_agent_callback` → invocation root span end.
- **OpenTelemetry W3C Trace Context**: Standard for propagating trace context
  across services via `traceparent` HTTP header.
- **Google Cloud Trace**: GCP-native tracing. Integrates with OpenTelemetry
  via OTLP exporter.

## Capabilities

- Creates OTel spans for each tool call within an ADK invocation.
- Propagates trace context (parent span ID) from invocation root to child spans.
- Correlates spans using `session_id` and `invocation_id` as span attributes.
- Exports traces to Google Cloud Trace via OTLP/gRPC.
- Traces multi-agent pipelines with parent-child span relationships.
- Records span events for exceptions and key state transitions.
- Implements span sampling for high-throughput production agents.
- Provides trace analysis in ADK Web UI Trace View for local debugging.

## Step-by-Step Instructions

1. **Use ADK Web UI Trace View** for local debugging (no setup required):
   ```bash
   adk web path/to/my_agent
   ```
   Click the **Trace** tab. Each user message groups all related events.
   Click trace rows to inspect Event, Request, Response, and Graph tabs.

2. **Set up OpenTelemetry** for distributed tracing:
   ```python
   from opentelemetry import trace
   from opentelemetry.sdk.trace import TracerProvider
   from opentelemetry.sdk.trace.export import BatchSpanProcessor
   from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

   provider = TracerProvider()
   provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:4317")))
   trace.set_tracer_provider(provider)
   tracer = trace.get_tracer("my_agent")
   ```

3. **Create an invocation root span** in `before_agent_callback`:
   ```python
   _active_spans = {}

   def start_invocation_span(callback_context: CallbackContext) -> None:
       span = tracer.start_span("agent_invocation")
       span.set_attribute("session.id", callback_context.session_id)
       span.set_attribute("user.id", callback_context.user_id)
       _active_spans[callback_context.session_id] = span
       return None
   ```

4. **Create child spans** in tool functions:
   ```python
   def my_tool(query: str, tool_context: ToolContext) -> dict:
       parent_span = _active_spans.get(tool_context.session_id)
       ctx = trace.set_span_in_context(parent_span) if parent_span else None
       with tracer.start_as_current_span("my_tool", context=ctx) as span:
           span.set_attribute("tool.name", "my_tool")
           span.set_attribute("query", query)
           result = perform_operation(query)
           return result
   ```

5. **End the root span** in `after_agent_callback`:
   ```python
   def end_invocation_span(callback_context: CallbackContext) -> None:
       span = _active_spans.pop(callback_context.session_id, None)
       if span:
           span.end()
       return None
   ```

6. **Export to Google Cloud Trace**:
   ```bash
   pip install opentelemetry-exporter-gcp-trace
   ```
   ```python
   from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
   provider.add_span_processor(BatchSpanProcessor(CloudTraceSpanExporter()))
   ```

## Input Format

```python
{
  "agent_id": "my_agent",
  "session_id": "sess_abc123",
  "tool_name": "search_products",
  "execution_time_ms": 342,
  "status": "success",
  "error": None,
  "metadata": {
    "invocation_id": "inv_7f8a9b",
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "parent_span_id": "00f067aa0ba902b7",
    "sampling_rate": 1.0
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
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "a2fb4a1d1a96d312",
  "parent_span_id": "00f067aa0ba902b7",
  "trace_backend": "google_cloud_trace"
}
```

## Error Handling

- **Missing parent span** for child tool calls: Create the tool span as a root
  span with `session.id` attribute for correlation. Never block tool execution
  waiting for a span.
- **OTel exporter connection failure**: `BatchSpanProcessor` buffers and retries.
  If the buffer fills and export still fails, spans are dropped. Log a warning.
- **Span context not propagated across sub-agents**: Use OTel's `context.attach()`
  to propagate context explicitly when calling sub-agents.
- **Trace IDs not matching across services**: Ensure W3C Trace Context headers
  are propagated. Use `TraceContextTextMapPropagator` for HTTP inter-service calls.
- **Missing traces for failed invocations**: Always call `span.record_exception(exc)`
  in except blocks and `span.end()` in finally blocks.

## Examples

### Example 1 — Complete invocation tracing with callbacks

```python
import time
from typing import Optional
from opentelemetry import trace
from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content

tracer = trace.get_tracer("my_agent")
_active_root_spans: dict = {}

def start_trace(callback_context: CallbackContext) -> Optional[Content]:
    """Starts root span for the agent invocation."""
    span = tracer.start_span("agent.invocation")
    span.set_attribute("session.id", callback_context.session_id)
    span.set_attribute("user.id", callback_context.user_id)
    _active_root_spans[callback_context.session_id] = (span, time.time())
    return None

def end_trace(callback_context: CallbackContext) -> Optional[Content]:
    """Ends root span and records total latency."""
    entry = _active_root_spans.pop(callback_context.session_id, None)
    if entry:
        span, start_ts = entry
        span.set_attribute("total_latency_ms", round((time.time() - start_ts) * 1000, 2))
        span.end()
    return None

agent = LlmAgent(
    name="traced_agent",
    model="gemini-2.0-flash",
    before_agent_callback=start_trace,
    after_agent_callback=end_trace,
)
```

### Example 2 — Tool span as child of invocation root

```python
from opentelemetry import trace, context
import time
from google.adk.tools import ToolContext

tracer = trace.get_tracer("my_agent")
_active_root_spans: dict = {}  # shared with callbacks above

def search_products(query: str, tool_context: ToolContext) -> dict:
    """
    Args:
        query (str): Search query.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'results' (list).
    """
    parent = _active_root_spans.get(tool_context.session_id)
    ctx = trace.set_span_in_context(parent[0]) if parent else None

    with tracer.start_as_current_span("tool.search_products", context=ctx) as span:
        span.set_attribute("session.id", tool_context.session_id)
        span.set_attribute("tool.args.query", query)

        start = time.time()
        try:
            results = _do_search(query)
            span.set_attribute("tool.status", "success")
            span.set_attribute("tool.result.count", len(results))
            span.set_attribute("tool.latency_ms", round((time.time() - start) * 1000, 2))
            return {"results": results}
        except Exception as exc:
            span.record_exception(exc)
            span.set_attribute("tool.status", "error")
            return {"error": str(exc), "error_type": type(exc).__name__}

def _do_search(query):
    return []
```

### Example 3 — Google Cloud Trace export

```python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter

def setup_cloud_trace():
    provider = TracerProvider()
    provider.add_span_processor(BatchSpanProcessor(CloudTraceSpanExporter()))
    from opentelemetry import trace
    trace.set_tracer_provider(provider)
    return trace.get_tracer("my_agent")
```

## Edge Cases

- **`ParallelAgent` sub-agents**: Each sub-agent runs concurrently. Create
  sibling child spans (same parent) — not sequential spans.
- **Long-running tool calls** (> 30s): Add span events at intermediate points:
  `span.add_event("data_fetched", {"row_count": 1500})`.
- **Session ID collision in `_active_root_spans`**: Use `invocation_id` (not
  `session_id`) as the span dict key when a session can have concurrent invocations.
- **Trace context loss on async tool calls**: Use `asyncio`-compatible OTel
  context propagation — `context.attach(ctx)` returns a token; call `context.detach(token)` in finally.

## Integration Notes

- **ADK Web UI Trace View**: The zero-setup local tracing tool. Blue rows
  indicate events; clicking opens Event/Request/Response/Graph tabs.
- **`google-adk-observability`**: This skill (tracing) is a sub-component of
  the full observability stack. Apply `observability` skill first.
- **`google-adk-performance-benchmarking`**: Uses span latency data as the
  primary source of tool execution timing for benchmarks.
- **Google Cloud Trace**: Use `CloudTraceSpanExporter` for GCP. Auto-integrates
  with Cloud Run and GKE when ADC credentials are configured.
