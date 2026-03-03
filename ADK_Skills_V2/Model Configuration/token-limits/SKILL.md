---
name: token-limits
description: >
  Use this skill when configuring max_output_tokens and managing input token
  budgets for a Google ADK LlmAgent. Covers setting max_output_tokens in
  GenerateContentConfig, estimating input token usage from session history
  and instructions, preventing context window overflow, and implementing
  token-budget-aware context management strategies.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: model-configuration
---

## Purpose

This skill configures and manages token limits for Google ADK `LlmAgent`. It
covers two dimensions of token management: (1) **output limit** — constraining
the model's maximum generation length via `max_output_tokens` in
`GenerateContentConfig`; and (2) **input budget** — monitoring and controlling
the total input tokens consumed by the session history, instruction, and injected
context to prevent context window overflow. Proper token management is critical
for controlling cost, latency, and reliability in production ADK deployments.

## When to Use

- When agent responses are excessively long and need to be capped.
- When the session history grows and risks exceeding the model's context window
  (1M tokens for Flash, 2M for Pro).
- When cost control requires limiting per-invocation token usage.
- When latency is high and reducing `max_output_tokens` would help.
- When the ADK agent raises a context window overflow error.
- When designing a long-running multi-turn agent that needs proactive token budget management.

## When NOT to Use

- Do not set `max_output_tokens` so low that the agent's responses are always
  truncated — this degrades user experience and causes incomplete tool results.
- Do not rely solely on `max_output_tokens` for cost control — monitor input
  tokens separately, as they often dominate in long-session agents.
- Do not discard session history without summarization — use the
  `google-adk-context-summarization` skill to compress history instead of dropping it.

## Google ADK Context

- **`GenerateContentConfig.max_output_tokens`**: Integer. Caps the model's
  generation length in tokens. ADK stops generation when this limit is reached.
- **Gemini context windows**:
  - `gemini-2.0-flash`: 1,000,000 input tokens
  - `gemini-2.0-flash-lite`: 1,000,000 input tokens
  - `gemini-2.0-pro`: 2,000,000 input tokens
- **Input token composition**: `total_input_tokens = instruction_tokens + session_history_tokens + tool_schema_tokens + injected_context_tokens`
- **`event.usage_metadata`**: ADK events carry `prompt_token_count` and
  `total_token_count` on LLM response events. Use to monitor per-invocation
  token usage.
- **Token budget warning threshold**: When `total_input_tokens > 0.8 × context_window`,
  trigger context compression or history pruning.
- **Tool schema tokens**: Each tool's schema is included in the context. Agents
  with many tools have higher baseline input token counts.

## Capabilities

- Caps response length via `max_output_tokens` in `GenerateContentConfig`.
- Monitors per-invocation input and output token counts from event metadata.
- Detects input token budget warning thresholds.
- Triggers context compression when session history approaches context window limit.
- Configures token limits per sub-agent in multi-agent pipelines.
- Reduces cost and latency by right-sizing output token limits.

## Step-by-Step Instructions

1. **Set `max_output_tokens`** based on expected response length:
   ```python
   from google.genai.types import GenerateContentConfig
   config = GenerateContentConfig(
       max_output_tokens=512,   # e.g., for chat responses
   )
   ```
   Common values:
   - Chat/support responses: 256–512 tokens
   - Analysis/summary: 512–1024 tokens
   - Long-form reports: 1024–4096 tokens

2. **Apply to agent**:
   ```python
   agent = LlmAgent(
       name="chat_agent",
       model="gemini-2.0-flash",
       generate_content_config=config,
   )
   ```

3. **Monitor input tokens** from response events:
   ```python
   for event in runner.run(...):
       if event.usage_metadata:
           input_tokens = event.usage_metadata.prompt_token_count
           output_tokens = event.usage_metadata.candidates_token_count
   ```

4. **Check budget in `before_agent_callback`** and warn/compress if needed:
   ```python
   CONTEXT_WINDOW = 1_000_000  # Flash
   WARNING_THRESHOLD = 0.80    # 80% of context window

   def check_token_budget(callback_context: CallbackContext):
       used = int(callback_context.state.get("session_total_tokens", 0))
       if used > CONTEXT_WINDOW * WARNING_THRESHOLD:
           callback_context.state["context_compression_needed"] = True
       return None
   ```

5. **Implement graduated output limits** per task type if the agent handles
   multiple response types.

6. **Log token usage** per invocation for cost tracking and trend analysis.

## Input Format

```python
{
  "model": "gemini-2.0-flash",
  "temperature": 0.3,
  "top_k": 40,
  "top_p": 0.95,
  "max_tokens": 512,
  "stop_sequences": [],
  "stream": False,
  "response_schema": {},
  "deterministic": False
}
```

## Output Format

```python
{
  "model_used": "gemini-2.0-flash",
  "configuration_applied": {"max_output_tokens": 512},
  "output_valid": True,
  "structured_output": {},
  "token_usage": {
    "input_tokens": 1250,
    "output_tokens": 380,
    "total_tokens": 1630,
    "context_window_used_pct": 0.16
  }
}
```

## Error Handling

- **Response truncated at `max_output_tokens`**: The model stops mid-sentence.
  Increase the limit or add: "Keep all responses under [N] words." in the instruction.
- **Context window overflow** (`INVALID_ARGUMENT: context window exceeded`):
  Apply context summarization immediately. Switch to a larger-context model.
- **Input tokens growing unboundedly** across multi-turn sessions: Implement
  a sliding window that keeps only the last N turns of history, or summarize
  older turns.
- **`usage_metadata` is None**: Only LLM response events carry this. Aggregate
  across multiple events per invocation.
- **Tool schema inflate token count unexpectedly**: Remove unused tools from the
  agent's tool list. Each tool's JSON schema is included in every LLM request.

## Examples

### Example 1 — Token-limited chat agent

```python
from google.adk.agents import LlmAgent
from google.genai.types import GenerateContentConfig

CHAT_CONFIG = GenerateContentConfig(
    temperature=0.4,
    max_output_tokens=400,   # Chat-appropriate response length
)

chat_agent = LlmAgent(
    name="support_chat_agent",
    model="gemini-2.0-flash",
    instruction="""
    You are a customer support agent. Keep responses concise and under 100 words.
    If more detail is needed, offer to elaborate.
    """,
    generate_content_config=CHAT_CONFIG,
    tools=[check_order, process_refund],
)
```

### Example 2 — Token budget monitor with before_agent_callback

```python
import json
import logging
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content, Part

logger = logging.getLogger("token_budget")

CONTEXT_WINDOW_TOKENS = 1_000_000   # gemini-2.0-flash
WARNING_THRESHOLD = 0.80             # Warn at 80%
CRITICAL_THRESHOLD = 0.95            # Hard limit at 95%

def token_budget_guard(callback_context: CallbackContext) -> Optional[Content]:
    """Checks session token usage and blocks invocations near the context limit."""
    used_tokens = int(callback_context.state.get("session_total_tokens", 0))
    used_pct = used_tokens / CONTEXT_WINDOW_TOKENS

    if used_pct >= CRITICAL_THRESHOLD:
        logger.error(json.dumps({
            "event": "context_window_critical",
            "session_id": callback_context.session_id,
            "used_pct": round(used_pct, 3),
        }))
        return Content(parts=[Part(text=(
            "This session has reached its context limit. "
            "Please start a new session to continue."
        ))])

    if used_pct >= WARNING_THRESHOLD:
        logger.warning(json.dumps({
            "event": "context_window_warning",
            "session_id": callback_context.session_id,
            "used_pct": round(used_pct, 3),
        }))
        callback_context.state["context_compression_needed"] = True

    return None
```

### Example 3 — Per-invocation token tracking from events

```python
from google.adk.runners import Runner
from google.genai.types import Content, Part

async def run_with_token_tracking(runner: Runner, user_id: str, session_id: str, query: str, session_service) -> dict:
    """Runs agent and tracks per-invocation token usage in session state."""
    msg = Content(parts=[Part(text=query)])
    invocation_input_tokens = 0
    invocation_output_tokens = 0
    response_text = ""

    for event in runner.run(user_id=user_id, session_id=session_id, new_message=msg):
        if event.usage_metadata:
            invocation_input_tokens = event.usage_metadata.prompt_token_count or 0
            invocation_output_tokens = event.usage_metadata.candidates_token_count or 0
        if event.is_final_response() and event.content and event.content.parts:
            response_text = event.content.parts[0].text or ""

    # Update cumulative session token count in state
    session = await session_service.get_session(app_name="my_app", user_id=user_id, session_id=session_id)
    if session:
        current_total = int(session.state.get("session_total_tokens", 0))
        # Note: In production, use session_service to append a state event
        # For simplicity, log here
        import logging
        logging.getLogger("token_tracker").info(
            f"Invocation tokens: input={invocation_input_tokens}, output={invocation_output_tokens}, "
            f"session_total={current_total + invocation_input_tokens + invocation_output_tokens}"
        )

    return {
        "response": response_text,
        "input_tokens": invocation_input_tokens,
        "output_tokens": invocation_output_tokens,
        "total_tokens": invocation_input_tokens + invocation_output_tokens,
    }
```

## Edge Cases

- **Structured output truncated**: If `max_output_tokens` is set too low, the
  JSON response may be truncated and fail Pydantic validation. Set `max_output_tokens`
  to at least 2x the expected schema JSON string length.
- **Tool results consume input tokens**: Long tool responses (e.g., a search
  returning 100 results) inflate input tokens on the next turn. Truncate tool
  results before storing in state.
- **Many tools = high baseline input tokens**: 10 tools with detailed docstrings
  may add 5000–15000 tokens to each request. Minimize tool count and docstring
  length for token-sensitive agents.
- **Output limit causes streaming to stop mid-sentence**: In streaming mode, the
  stream is cut at the token limit. Implement graceful streaming termination in
  the event consumer.

## Integration Notes

- **`google-adk-cost-tracking`**: Token counts are the primary input to cost
  estimation. This skill provides the monitoring infrastructure; cost-tracking
  skill computes the dollar amount.
- **`google-adk-context-summarization`**: When session history approaches the
  context window limit, apply context summarization to compress old turns.
- **`google-adk-latency-optimization`**: Reducing `max_output_tokens` reduces
  streaming latency proportionally. A key latency optimization for chat agents.
- **`google-adk-streaming-responses`**: `max_output_tokens` applies to streaming
  the same way as non-streaming. The stream is cut at the token limit.
