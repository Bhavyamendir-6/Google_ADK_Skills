---
name: google-adk-cost-tracking
description: >
  Use this skill when a Google ADK agent needs to track LLM token usage, tool
  execution costs, and total per-invocation cost. Covers reading token metadata
  from ADK event objects, computing cost estimates from Gemini pricing,
  accumulating cost in session.state, setting budget limits, and reporting
  cost metrics to Google Cloud Monitoring or custom dashboards.
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

This skill tracks the computational and financial cost of Google ADK agent
invocations. It captures LLM token usage from ADK event metadata, computes
cost estimates using Gemini API pricing, accumulates costs in `session.state`,
enforces per-session token budgets, and reports cost telemetry to monitoring
systems. This enables cost visibility, budget enforcement, and cost-per-user
attribution for production ADK deployments.

## When to Use

- When running ADK agents in production and needing per-invocation or per-session
  cost visibility.
- When implementing per-user or per-tenant cost budgets and enforcement.
- When analyzing which agent workflows or tools are most expensive.
- When cost is a factor in choosing between model variants (e.g., `gemini-flash`
  vs `gemini-pro`).
- When generating cost reports for ADK agent deployments in GCP billing dashboards.

## When NOT to Use

- Do not use this skill to enforce hard usage limits on safety-critical agents —
  a cost limit that aborts a safety check is dangerous.
- Do not rely solely on estimated costs — always cross-check with GCP billing
  for exact charges.
- Do not track cost at the `temp:` state key level — use session-scoped or
  `user:` scoped keys for accumulation.

## Google ADK Context

- **`event.usage_metadata`**: ADK `Event` objects from LLM calls contain
  `usage_metadata` with `prompt_token_count`, `candidates_token_count`, and
  `total_token_count`. Available on `final_response` events.
- **`tool_context.state`**: Use to accumulate running cost totals per session:
  `session_total_tokens`, `session_estimated_cost_usd`.
- **`tool_context.user_id`**: Use for `user:` prefixed cost keys for cross-session
  cost tracking.
- **`after_agent_callback`**: Primary hook for capturing post-invocation token
  usage from the final response event.
- **Gemini API Pricing** (approximate, verify at cloud.google.com/vertex-ai/pricing):
  - `gemini-2.0-flash`: ~$0.075 / 1M input tokens, ~$0.30 / 1M output tokens
  - `gemini-2.0-pro`: ~$1.25 / 1M input tokens, ~$5.00 / 1M output tokens
- **ADK Runner events**: The `Runner.run()` generator yields `Event` objects.
  Each `Event` with `author == "model"` and `is_final_response() == True` contains
  the token usage for that LLM call.

## Capabilities

- Reads per-invocation LLM token counts from ADK event `usage_metadata`.
- Computes estimated USD cost per invocation using configurable Gemini pricing.
- Accumulates per-session token totals in `session.state`.
- Accumulates per-user total cost in `user:` scoped state.
- Enforces per-session token budget limits — returns early if budget exceeded.
- Emits cost metrics to OpenTelemetry / Google Cloud Monitoring.
- Generates per-user and per-agent cost reports.

## Step-by-Step Instructions

1. **Capture token usage** from ADK runner events after each invocation:
   ```python
   for event in runner.run(...):
       if event.is_final_response() and event.usage_metadata:
           usage = event.usage_metadata
           input_tokens = usage.prompt_token_count
           output_tokens = usage.candidates_token_count
           total_tokens = usage.total_token_count
   ```

2. **Compute estimated cost** using Gemini pricing constants:
   ```python
   GEMINI_FLASH_PRICING = {
       "input_per_1m": 0.075,
       "output_per_1m": 0.30,
   }
   def estimate_cost(input_tokens: int, output_tokens: int, pricing: dict) -> float:
       return (input_tokens * pricing["input_per_1m"] / 1_000_000 +
               output_tokens * pricing["output_per_1m"] / 1_000_000)
   ```

3. **Accumulate cost in `session.state`** using `after_agent_callback`:
   ```python
   def track_cost_callback(callback_context: CallbackContext) -> None:
       # Token data captured from final event
       session_cost = callback_context.state.get("session_estimated_cost_usd", 0.0)
       callback_context.state["session_estimated_cost_usd"] = session_cost + invocation_cost
   ```

4. **Enforce a per-session budget** in `before_agent_callback`:
   ```python
   def enforce_budget(callback_context: CallbackContext):
       budget = 0.10  # $0.10 per session
       spent = callback_context.state.get("session_estimated_cost_usd", 0.0)
       if spent >= budget:
           from google.genai.types import Content, Part
           return Content(parts=[Part(text="Session budget exceeded. Please start a new session.")])
       return None
   ```

5. **Accumulate per-user cost** using `user:` scoped state:
   ```python
   user_cost = tool_context.state.get("user:total_cost_usd", 0.0)
   tool_context.state["user:total_cost_usd"] = user_cost + invocation_cost
   ```

6. **Emit cost metrics to OpenTelemetry**:
   ```python
   token_counter.add(total_tokens, {"agent": "my_agent", "model": "gemini-flash"})
   cost_histogram.record(invocation_cost * 100, {"agent": "my_agent"})  # in cents
   ```

7. **Generate cost reports** by querying `user:total_cost_usd` from sessions.

## Input Format

```python
{
  "agent_id": "my_agent",
  "session_id": "sess_abc123",
  "tool_name": None,
  "execution_time_ms": 820,
  "status": "success",
  "error": None,
  "metadata": {
    "model": "gemini-2.0-flash",
    "input_tokens": 1250,
    "output_tokens": 380,
    "total_tokens": 1630,
    "per_session_budget_usd": 0.10
  }
}
```

## Output Format

```python
{
  "metric_recorded": True,
  "evaluation_result": "pass",
  "latency_ms": 820,
  "cost_estimate": 0.000208,
  "issues_detected": [],
  "token_usage": {
    "input_tokens": 1250,
    "output_tokens": 380,
    "total_tokens": 1630
  },
  "session_total_cost_usd": 0.000832,
  "budget_remaining_usd": 0.099168,
  "budget_exceeded": False
}
```

## Error Handling

- **`usage_metadata` is None** (non-LLM events): Skip cost calculation. Only
  LLM response events carry `usage_metadata`.
- **Token count is zero**: The model returned a cached or empty response.
  Log a warning and skip accumulation.
- **Budget enforcement too aggressive** (blocks legitimate queries): Set budget
  to a high value (e.g., `$1.00`) initially, monitor actual consumption for 1
  week, then tighten.
- **OTel metric emission failure**: Catch and log. Never block agent execution
  on metric emission failures.
- **`user:total_cost_usd` type mismatch** (stored as string): Validate type on
  read: `float(tool_context.state.get("user:total_cost_usd", 0.0))`.
- **Pricing constants out of date**: Store pricing in `app:` scoped state for
  easy update without redeployment.

## Examples

### Example 1 — Full cost tracking with after_agent_callback

```python
import json
import logging
from typing import Optional
from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content

logger = logging.getLogger("cost_tracker")

# Gemini pricing (USD per 1M tokens)
PRICING = {
    "gemini-2.0-flash": {"input_per_1m": 0.075, "output_per_1m": 0.30},
    "gemini-2.0-pro":   {"input_per_1m": 1.25,  "output_per_1m": 5.00},
}

_last_invocation_tokens: dict = {}  # session_id → {input, output, total}

def capture_cost_after_agent(callback_context: CallbackContext) -> Optional[Content]:
    """Records token usage and estimated cost after each agent invocation."""
    sid = callback_context.session_id
    token_data = _last_invocation_tokens.pop(sid, None)
    if not token_data:
        return None

    model = callback_context.state.get("meta:model", "gemini-2.0-flash")
    pricing = PRICING.get(model, PRICING["gemini-2.0-flash"])

    input_tokens = token_data.get("input", 0)
    output_tokens = token_data.get("output", 0)
    est_cost = (input_tokens * pricing["input_per_1m"] / 1_000_000 +
                output_tokens * pricing["output_per_1m"] / 1_000_000)

    # Accumulate session cost
    session_cost = float(callback_context.state.get("session_estimated_cost_usd", 0.0))
    callback_context.state["session_estimated_cost_usd"] = round(session_cost + est_cost, 8)
    callback_context.state["session_total_tokens"] = (
        int(callback_context.state.get("session_total_tokens", 0)) + token_data.get("total", 0)
    )

    # Accumulate user lifetime cost
    user_cost = float(callback_context.state.get("user:lifetime_cost_usd", 0.0))
    callback_context.state["user:lifetime_cost_usd"] = round(user_cost + est_cost, 8)

    logger.info(json.dumps({
        "event": "cost_recorded",
        "session_id": sid,
        "user_id": callback_context.user_id,
        "model": model,
        "input_tokens": input_tokens,
        "output_tokens": output_tokens,
        "invocation_cost_usd": round(est_cost, 8),
        "session_total_cost_usd": round(session_cost + est_cost, 8),
    }))
    return None


def enforce_session_budget(callback_context: CallbackContext) -> Optional[Content]:
    """Blocks invocations that exceed the per-session cost budget."""
    budget = float(callback_context.state.get("app:session_budget_usd", 0.10))
    spent = float(callback_context.state.get("session_estimated_cost_usd", 0.0))
    if spent >= budget:
        return Content(parts=[{"text": (
            f"Session budget of ${budget:.2f} reached (spent: ${spent:.4f}). "
            "Please start a new session to continue."
        )}])
    return None


agent = LlmAgent(
    name="cost_tracked_agent",
    model="gemini-2.0-flash",
    before_agent_callback=enforce_session_budget,
    after_agent_callback=capture_cost_after_agent,
)
```

### Example 2 — Capturing token usage from runner events

```python
from google.adk.runners import Runner
from google.genai.types import Content, Part

def run_and_track_cost(runner: Runner, user_id: str, session_id: str, query: str) -> dict:
    """Runs the agent and tracks token usage from the runner event stream."""
    msg = Content(parts=[Part(text=query)])
    token_data = {}
    response_text = ""

    for event in runner.run(user_id=user_id, session_id=session_id, new_message=msg):
        if event.is_final_response():
            if event.usage_metadata:
                token_data = {
                    "input": event.usage_metadata.prompt_token_count or 0,
                    "output": event.usage_metadata.candidates_token_count or 0,
                    "total": event.usage_metadata.total_token_count or 0,
                }
            if event.content and event.content.parts:
                response_text = event.content.parts[0].text or ""

    # Store for callback consumption
    _last_invocation_tokens[session_id] = token_data

    pricing = PRICING["gemini-2.0-flash"]
    est_cost = (token_data.get("input", 0) * pricing["input_per_1m"] / 1_000_000 +
                token_data.get("output", 0) * pricing["output_per_1m"] / 1_000_000)

    return {
        "response": response_text,
        "token_usage": token_data,
        "estimated_cost_usd": round(est_cost, 8),
    }
```

### Example 3 — Cost metric emission to OpenTelemetry

```python
from opentelemetry import metrics

meter = metrics.get_meter("my_agent.cost")
token_counter = meter.create_counter("adk.agent.tokens.total", unit="tokens")
cost_gauge = meter.create_histogram("adk.agent.cost_cents", unit="cents")

def emit_cost_metrics(model: str, token_data: dict, est_cost_usd: float):
    """Emits token and cost metrics to OpenTelemetry."""
    attrs = {"model": model}
    token_counter.add(token_data.get("total", 0), attrs)
    cost_gauge.record(round(est_cost_usd * 100, 4), attrs)  # Convert to cents
```

## Edge Cases

- **Multi-turn session cost accumulation**: `session_estimated_cost_usd` grows
  across all turns. Provide users with a mid-session cost report if it exceeds
  50% of budget.
- **Sub-agent token usage**: Each sub-agent in a pipeline makes its own LLM
  call. Ensure all sub-agents' callbacks are wired up for cost tracking.
- **Free tier / gratis tokens**: Some Vertex AI usage may be under free tier.
  Tag free-tier usage separately to avoid misleading cost reports.
- **Pricing changes from Google**: Store pricing in `app:gemini_flash_input_per_1m`
  state so it can be updated without redeployment.
- **Zero output tokens** (model refused to generate): Still charge input tokens.
  Log refusals separately for quality analysis.

## Integration Notes

- **`google-adk-latency-optimization`**: Model downgrade from `gemini-pro` to
  `gemini-flash` dramatically reduces both latency and cost. Track both together.
- **`google-adk-observability`**: Cost metrics are emitted as OTel counters and
  histograms. Set up dashboards for `adk.agent.cost_cents` in Cloud Monitoring.
- **GCP Billing**: For exact costs, use Google Cloud Billing export to BigQuery.
  This skill provides estimates — cross-check with actual billing monthly.
- **`google-adk-session-management`**: Session service must be persistent to
  retain `session_estimated_cost_usd` across server restarts.
