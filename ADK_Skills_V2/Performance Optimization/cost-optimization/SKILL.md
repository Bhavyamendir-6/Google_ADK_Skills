---
name: cost-optimization
description: >
  Use this skill when reducing the operational cost of running Google ADK agent
  pipelines in production. Covers selecting cost-efficient model variants,
  monitoring per-invocation cost from token usage metadata, applying caching to
  avoid redundant API calls, routing simple queries to smaller models, and
  implementing cost budgets with circuit breakers.
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

This skill reduces the operational cost of Google ADK agent deployments. Cost
is primarily driven by token consumption (input × input_price + output × output_price)
across all LlmAgent invocations. Secondary costs include vector store API calls,
external tool API fees, and compute infrastructure. This skill audits cost drivers,
applies model-level cost reduction (model downgrade, output token capping), applies
caching to avoid redundant calls, and implements cost budgets with automatic
circuit breakers to prevent runaway spend.

## When to Use

- When monthly inference costs exceed budget targets.
- When per-session or per-user cost analysis reveals inefficiency.
- When expensive Pro model is used for tasks that Flash or Flash-Lite can handle.
- When the same expensive tool call (e.g., web search API) is repeated redundantly.
- When high-volume use cases require cost reduction to be economically viable.
- When implementing cost-per-user budgets and enforcement.

## When NOT to Use

- Do not optimize cost at the expense of critical quality — always validate with eval.
- Do not downgrade the model for tasks genuinely requiring Pro-level reasoning.
- Do not apply aggressive caching for user-specific personalized responses where
  stale results would be incorrect.

## Google ADK Context

- **Gemini pricing (approximate, as of 2025)**:
  - `gemini-2.0-flash-lite`: ~$0.075/1M input tokens, ~$0.30/1M output tokens.
  - `gemini-2.0-flash`: ~$0.075/1M input tokens, ~$0.30/1M output tokens.
  - `gemini-2.0-pro`: ~$1.25/1M input tokens, ~$5.00/1M output tokens (estimated).
- **`event.usage_metadata`**: Provides `prompt_token_count` and `candidates_token_count`
  per invocation. Use for per-session cost accounting.
- **Model routing**: Use a Flash-Lite classifier to route queries to the appropriate
  model — cheap queries → Flash-Lite; complex queries → Flash or Pro.
- **Tool result caching**: Cache expensive external API results in `tool_context.state`
  with TTL to avoid redundant paid API calls.
- **`max_output_tokens`**: Reducing output tokens directly reduces output cost.
- **Circuit breaker pattern**: When session cost exceeds a per-session budget,
  block further invocations until the session budget resets.

## Capabilities

- Tracks per-invocation and per-session cost from `event.usage_metadata`.
- Routes queries to the most cost-efficient model via a classifier.
- Applies `max_output_tokens` budgets to reduce output cost.
- Caches external API results to avoid redundant paid calls.
- Implements per-session and per-user cost budgets with circuit breakers.
- Generates cost reports by user, session, and model.

## Step-by-Step Instructions

1. **Track per-invocation cost** from `event.usage_metadata`:
   ```python
   PRICE_PER_1M_INPUT = 0.075   # Flash-Lite
   PRICE_PER_1M_OUTPUT = 0.300
   for event in runner.run(...):
       if event.usage_metadata:
           input_cost = (event.usage_metadata.prompt_token_count / 1_000_000) * PRICE_PER_1M_INPUT
           output_cost = (event.usage_metadata.candidates_token_count / 1_000_000) * PRICE_PER_1M_OUTPUT
           total_invocation_cost = input_cost + output_cost
   ```

2. **Apply cost budget to session state**:
   ```python
   session_cost = float(tool_context.state.get("user:session_cost_usd", 0)) + total_invocation_cost
   tool_context.state["user:session_cost_usd"] = session_cost
   ```

3. **Implement session cost circuit breaker** in `before_agent_callback`:
   ```python
   SESSION_COST_LIMIT_USD = 0.50
   cost = float(callback_context.state.get("user:session_cost_usd", 0))
   if cost >= SESSION_COST_LIMIT_USD:
       return Content(parts=[Part(text="Session budget reached. Please start a new session.")])
   ```

4. **Route to cheaper model** using a classifier:
   ```python
   classifier = LlmAgent(name="router", model="gemini-2.0-flash-lite",
                          instruction="Classify query as: simple | complex",
                          output_schema=QueryType, output_key="query_type")
   ```

5. **Cache expensive results** with TTL (see `google-adk-caching-strategies`).

6. **Downgrade model** for simple classification/extraction sub-agents:
   ```python
   sub_agent = LlmAgent(model="gemini-2.0-flash-lite", ...)
   ```

7. **Run eval** on the downgraded model to validate quality before deployment.

## Input Format

```python
{
  "operation": "complex_query",
  "execution_time_ms": 3800,
  "token_usage": 12000,
  "memory_usage_mb": 64,
  "cost_estimate": 0.0042,
  "retrieval_latency_ms": 200,
  "cache_available": True,
  "parallelizable": False,
  "context_size_tokens": 12000
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "model_routing + caching + output_token_cap",
  "performance_improvement": {
    "latency_reduction_ms": 800,
    "token_reduction": 5000,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.0025,
    "throughput_improvement": 1.0
  }
}
```

## Error Handling

- **Cost tracking breaks when `usage_metadata` is null**: Only LLM response events
  carry metadata. Filter events: `if event.usage_metadata and event.usage_metadata.prompt_token_count`.
- **Circuit breaker blocks legitimate requests**: Set the cost limit conservatively.
  Implement a reset mechanism (e.g., per-day budget resets at midnight UTC).
- **Flash-Lite produces wrong answers**: Run quality eval. If quality insufficient,
  use Flash. Document which task types require Pro vs. Flash vs. Flash-Lite.
- **Tool API costs not tracked** (e.g., search API charges per call): Track API
  calls separately. Increment `user:api_call_count` in tool functions.

## Examples

### Example 1 — Cost tracking and circuit breaker

```python
import time
import logging
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content, Part

logger = logging.getLogger("cost_tracker")

MODEL_PRICING = {
    "gemini-2.0-flash-lite": {"input": 0.075, "output": 0.30},
    "gemini-2.0-flash":      {"input": 0.075, "output": 0.30},
    "gemini-2.0-pro":        {"input": 1.25,  "output": 5.00},
}
SESSION_BUDGET_USD = 0.50
DAILY_USER_BUDGET_USD = 5.00

def track_and_guard_cost(callback_context: CallbackContext) -> Optional[Content]:
    """Tracks session cost and enforces budget limits."""
    session_cost = float(callback_context.state.get("session_cost_usd", 0))
    user_daily_cost = float(callback_context.state.get("user:daily_cost_usd", 0))

    if session_cost >= SESSION_BUDGET_USD:
        logger.warning(f"Session budget exceeded: ${session_cost:.4f}")
        return Content(parts=[Part(text=(
            f"This session has reached its ${SESSION_BUDGET_USD:.2f} cost limit. "
            "Please start a new session to continue."
        ))])

    if user_daily_cost >= DAILY_USER_BUDGET_USD:
        logger.warning(f"Daily user budget exceeded: ${user_daily_cost:.4f}")
        return Content(parts=[Part(text=(
            "Your daily usage limit has been reached. Please try again tomorrow."
        ))])
    return None


def update_cost_tracking(input_tokens: int, output_tokens: int, model: str,
                           callback_context: CallbackContext):
    """Updates cost tracking state after each invocation."""
    pricing = MODEL_PRICING.get(model, MODEL_PRICING["gemini-2.0-flash"])
    inv_cost = (input_tokens / 1_000_000 * pricing["input"] +
                output_tokens / 1_000_000 * pricing["output"])
    callback_context.state["session_cost_usd"] = (
        float(callback_context.state.get("session_cost_usd", 0)) + inv_cost
    )
    callback_context.state["user:daily_cost_usd"] = (
        float(callback_context.state.get("user:daily_cost_usd", 0)) + inv_cost
    )
    logger.info(f"Invocation cost: ${inv_cost:.6f}. Session total: ${callback_context.state['session_cost_usd']:.4f}")
```

### Example 2 — Model routing by query complexity

```python
from pydantic import BaseModel
from typing import Literal
from google.adk.agents import LlmAgent, SequentialAgent
from google.genai.types import GenerateContentConfig

class QueryComplexity(BaseModel):
    complexity: Literal["simple", "moderate", "complex"]

# Flash-Lite classifier (very cheap)
complexity_classifier = LlmAgent(
    name="complexity_router",
    model="gemini-2.0-flash-lite",
    instruction="Classify the user query complexity: simple (factual/short), moderate (analysis), complex (multi-step reasoning).",
    generate_content_config=GenerateContentConfig(temperature=0, max_output_tokens=50),
    output_schema=QueryComplexity,
    output_key="query_complexity",
)

# Responder selects model based on classification
responder = LlmAgent(
    name="adaptive_responder",
    model="gemini-2.0-flash",  # Default to Flash; consider dynamic model selection
    instruction="Complexity: {query_complexity?}. Provide an appropriate response.",
    tools=[search_flights, get_weather, process_payment],
)

pipeline = SequentialAgent(name="cost_optimized_pipeline", sub_agents=[complexity_classifier, responder])
```

## Edge Cases

- **Budget reset mechanism**: Store `user:daily_cost_reset_date` and reset
  `user:daily_cost_usd` to 0 when the date changes.
- **High-volume burst** (1000 requests/minute): Implement rate limiting at the
  API gateway level in addition to per-session budgets.
- **Shared app: state cost tracking** across users: Track per-user cost in
  `user:` state; track aggregate in `app:` state for account-level billing.

## Integration Notes

- **`google-adk-token-usage-optimization`**: Reducing tokens is the primary cost lever.
- **`google-adk-caching-strategies`**: Cached results cost $0 in inference.
- **`google-adk-model-selection-optimization`**: Model downgrade is the
  highest-impact cost reduction strategy (10x between Flash-Lite and Pro).
- **`google-adk-observability`**: Cost tracking data feeds into dashboards and alerts.
