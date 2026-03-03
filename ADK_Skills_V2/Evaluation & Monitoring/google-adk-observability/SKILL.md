---
name: google-adk-observability
description: >
  Use this skill when a Google ADK agent requires full-stack observability
  including logs, traces, metrics, and reasoning inspection. Covers ADK's
  observability architecture, integration with OpenTelemetry, Arize Phoenix,
  and Google Cloud Monitoring, and implementing observability hooks in ADK
  agents for production monitoring and debugging.
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

This skill implements comprehensive observability for Google ADK agents —
spanning logs (what happened), traces (how it happened), and metrics (how well
it performed). ADK provides configurable logging and integrates with external
observability tools (OpenTelemetry, Arize Phoenix) for advanced monitoring and
analysis. This skill enables real-time visibility into agent reasoning, tool
invocations, session lifecycles, and model interactions in production.

## When to Use

- When deploying an ADK agent to production and needing end-to-end visibility.
- When integrating ADK agents with an existing OpenTelemetry-based observability
  stack.
- When using Arize Phoenix or another LLM observability platform for ADK agent
  analysis.
- When setting up dashboards for agent health, latency, and error rates.
- When debugging complex multi-agent execution flows that span multiple tool
  calls and sub-agents.

## When NOT to Use

- Do not enable full trace collection in extremely high-throughput production
  agents without sampling — telemetry overhead can degrade performance.
- Do not rely on observability tools as the primary debugging mechanism for local
  development — use ADK Web UI Trace View for interactive debugging.
- Do not store full LLM request/response payloads in observability backends
  without PII review and data governance approval.

## Google ADK Context

- **ADK Observability Model**: ADK measures internal state via reasoning traces,
  tool calls, and model outputs analyzed through external telemetry and structured
  logs.
- **ADK configurable logging**: Foundation layer. All ADK events are emittable
  as structured logs via Python's `logging` module.
- **ADK Web UI Trace View**: Built-in trace viewer. Shows event groupings,
  model requests/responses, tool call graphs. Available at `adk web <agent_path>`.
- **ADK Integrations for observability**: Listed at
  `https://google.github.io/adk-docs/integrations/?topic=observability`.
  Key integrations:
  - **Arize Phoenix**: LLM observability platform with ADK support.
  - **OpenTelemetry**: Vendor-neutral observability framework.
  - **Google Cloud Monitoring**: GCP-native metrics and alerting.
- **`tool_context.session_id`** / **`tool_context.user_id`**: Used as trace
  attributes and metric labels.
- **Agent Execution Lifecycle**: Observable at every step — invocation start,
  LLM request/response, tool call, tool result, final response.

## Capabilities

- Implements OpenTelemetry tracing for ADK agent tool calls and LLM invocations.
- Integrates Arize Phoenix for LLM-specific observability (hallucination, latency).
- Records custom metrics (tool latency, error rate, token usage) via OpenTelemetry.
- Creates Google Cloud Monitoring dashboards for production agents.
- Implements observability hooks in `before_agent_callback` and `after_agent_callback`.
- Provides trace correlation via `session_id` as a span attribute.
- Samples telemetry to manage overhead in high-throughput agents.

## Step-by-Step Instructions

1. **Select observability backend** based on deployment:
   - GCP deployment → Google Cloud Monitoring + Cloud Trace
   - Cross-platform → OpenTelemetry Collector
   - LLM-specific analysis → Arize Phoenix

2. **Install OpenTelemetry dependencies**:
   ```bash
   pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp
   ```

3. **Configure OpenTelemetry tracer**:
   ```python
   from opentelemetry import trace
   from opentelemetry.sdk.trace import TracerProvider
   from opentelemetry.sdk.trace.export import BatchSpanProcessor
   from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

   provider = TracerProvider()
   provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
   trace.set_tracer_provider(provider)
   tracer = trace.get_tracer("my_agent")
   ```

4. **Wrap tool functions with spans**:
   ```python
   def my_tool(query: str, tool_context: ToolContext) -> dict:
       with tracer.start_as_current_span("my_tool") as span:
           span.set_attribute("session.id", tool_context.session_id)
           span.set_attribute("user.id", tool_context.user_id)
           span.set_attribute("tool.name", "my_tool")
           result = perform_operation(query)
           span.set_attribute("tool.status", "success")
           return result
   ```

5. **Record custom metrics**:
   ```python
   from opentelemetry import metrics
   meter = metrics.get_meter("my_agent")
   tool_latency = meter.create_histogram("tool.latency.ms")
   tool_latency.record(342, {"tool": "my_tool", "status": "success"})
   ```

6. **Integrate Arize Phoenix** for LLM observability:
   ```bash
   pip install arize-phoenix-otel
   ```
   ```python
   from phoenix.otel import register
   tracer_provider = register(project_name="my_adk_agent", endpoint="http://localhost:6006/v1/traces")
   ```

7. **Set up sampling** for high-throughput:
   ```python
   from opentelemetry.sdk.trace.sampling import TraceIdRatioBased
   provider = TracerProvider(sampler=TraceIdRatioBased(0.1))  # 10% sampling
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
    "otel_endpoint": "http://collector:4317",
    "sampling_rate": 0.1,
    "environment": "production"
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
  "span_id": "00f067aa0ba902b7",
  "observability_backend": "opentelemetry"
}
```

## Error Handling

- **OpenTelemetry collector unavailable**: Spans are buffered by `BatchSpanProcessor`.
  If the collector is down, spans are dropped after the buffer fills. Configure
  `MaxQueueSize` and `ExportTimeout` to manage this gracefully.
- **Missing `session_id` in span context**: Default attribute to `"unknown_session"`.
  Log a warning immediately.
- **Arize Phoenix endpoint unavailable**: Catch connection errors. Fall back to
  logging-only mode — never let observability failures crash the agent.
- **Telemetry overhead too high**: Reduce sampling rate from 1.0 to 0.1.
  Use `BatchSpanProcessor` instead of `SimpleSpanProcessor`.
- **PII in trace attributes**: Implement a `SpanProcessor` that scrubs known
  PII patterns from span attributes before export.

## Examples

### Example 1 — Full OpenTelemetry setup for ADK agent

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

def setup_observability(sampling_rate: float = 1.0, otel_endpoint: str = "http://localhost:4317"):
    """Configures OpenTelemetry for ADK agent observability."""
    # Traces
    trace_provider = TracerProvider(sampler=TraceIdRatioBased(sampling_rate))
    trace_provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint=otel_endpoint)))
    trace.set_tracer_provider(trace_provider)

    # Metrics
    metric_reader = PeriodicExportingMetricReader(OTLPMetricExporter(endpoint=otel_endpoint))
    metrics.set_meter_provider(MeterProvider(metric_readers=[metric_reader]))

    return trace.get_tracer("my_agent"), metrics.get_meter("my_agent")

tracer, meter = setup_observability(sampling_rate=0.1)
tool_latency_histogram = meter.create_histogram("adk.tool.latency_ms", unit="ms")
tool_error_counter = meter.create_counter("adk.tool.errors")
```

### Example 2 — Observed tool function with OTel span

```python
import time
from opentelemetry import trace
from google.adk.tools import ToolContext

tracer = trace.get_tracer("my_agent")

def search_products(query: str, tool_context: ToolContext) -> dict:
    """Searches the product catalog with full OTel observability.

    Args:
        query (str): Search query.
        tool_context (ToolContext): ADK tool context.

    Returns:
        dict: Search results.
    """
    with tracer.start_as_current_span("search_products") as span:
        span.set_attribute("session.id", tool_context.session_id)
        span.set_attribute("user.id", tool_context.user_id)
        span.set_attribute("tool.name", "search_products")
        span.set_attribute("query", query)

        start = time.time()
        try:
            results = _do_search(query)
            latency_ms = round((time.time() - start) * 1000, 2)
            span.set_attribute("tool.status", "success")
            span.set_attribute("result.count", len(results))
            span.set_attribute("latency_ms", latency_ms)
            return {"results": results}
        except Exception as exc:
            latency_ms = round((time.time() - start) * 1000, 2)
            span.record_exception(exc)
            span.set_attribute("tool.status", "error")
            span.set_attribute("latency_ms", latency_ms)
            return {"error": str(exc), "error_type": type(exc).__name__}

def _do_search(query):
    return []
```

### Example 3 — Arize Phoenix integration

```python
from phoenix.otel import register
from opentelemetry import trace

def setup_phoenix_observability():
    """Configures Arize Phoenix for LLM observability."""
    tracer_provider = register(
        project_name="my_adk_agent",
        endpoint="http://localhost:6006/v1/traces",
    )
    return trace.get_tracer("my_agent")
```

## Edge Cases

- **Multi-agent spans** (parent → sub-agent): Propagate OTel context through
  ADK sub-agent calls. The parent span context is automatically propagated within
  the same process via OTel's `Context` mechanism.
- **High cardinality attributes**: Avoid using raw user queries as span
  attributes at scale — they create high cardinality that inflates storage costs.
  Hash or categorize them instead.
- **Missing traces for failed invocations**: Ensure `span.record_exception()`
  is called in except blocks so failed spans are not silently dropped.
- **Observability backend downtime**: Use `BatchSpanProcessor` with a large
  queue (`MaxQueueSize=10000`) and retry export  — spans are buffered.

## Integration Notes

- **`google-adk-logging`**: Logging is the foundation layer. Observability
  tools consume structured logs as events. Set up logging before observability.
- **`google-adk-tracing`**: Detailed span-level tracing. This skill (`observability`)
  covers the overall stack; `tracing` covers span design and correlation.
- **Arize Phoenix**: ADK-compatible LLM observability platform. See
  `https://google.github.io/adk-docs/integrations/?topic=observability`.
- **Google Cloud Monitoring**: Use `google-cloud-monitoring` library to create
  custom metrics dashboards for ADK agent KPIs.
- **ADK Web UI Trace View**: Built-in trace visualization. Use it for local
  debugging; use OTel/Phoenix for production-scale observability.
