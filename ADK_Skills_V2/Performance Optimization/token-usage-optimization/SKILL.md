---
name: token-usage-optimization
description: >
  Use this skill when reducing the total token consumption of a Google ADK
  LlmAgent to lower inference costs and stay within context window limits.
  Covers optimizing instruction length, minimizing unnecessary state injection,
  compressing tool result payloads before storage, reducing conversation history
  token footprint via summarization, and right-sizing max_output_tokens.
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

This skill reduces the total token consumption of Google ADK `LlmAgent` invocations
to minimize inference costs and prevent context window overflow. Tokens are consumed
by: the system instruction, injected state values (`{key?}`), tool JSON schemas,
conversation history events, the user message, and model output. Reducing any of
these components lowers both cost and latency. This skill audits each token
source, applies targeted reduction strategies, and monitors token usage from
`event.usage_metadata` to validate improvements.

## When to Use

- When per-invocation token cost is too high (visible in cost tracking or `usage_metadata`).
- When the instruction is verbose and can be shortened without losing agent quality.
- When many `{key?}` injections inflate the context unnecessarily.
- When tool schemas are large (tools with many parameters and long docstrings).
- When conversation history grows across turns and dominates input token count.
- When tool results returned to the agent are large and can be summarized before injection.

## When NOT to Use

- Do not reduce instruction length to the point of degrading agent behavior —
  always validate quality with eval after reduction.
- Do not remove `{key?}` injections that are critically needed for agent reasoning.
- Do not reduce `max_output_tokens` below the minimum needed for complete responses.

## Google ADK Context

- **`event.usage_metadata.prompt_token_count`**: Input tokens per invocation.
  Sum across all events per turn.
- **`event.usage_metadata.candidates_token_count`**: Output tokens per invocation.
- **Token sources in order of typical impact**:
  1. Conversation history (grows unboundedly without management).
  2. Tool schemas (each tool adds its JSON schema on every request).
  3. System instruction (often inflated with redundant text).
  4. Injected state (`{key?}` with large values).
  5. User message (cannot be controlled).
- **`max_output_tokens`**: Caps output tokens. Set conservatively for agents
  with predictably short responses.
- **Tool docstrings**: The docstring of each tool function is included in the
  tool schema. Keep docstrings concise.
- **`tool_context.state` injection**: Each `{key?}` adds the value's token length
  to the instruction. Only inject what is needed for the current turn.

## Capabilities

- Audits token usage by source from `event.usage_metadata`.
- Reduces instruction token count while preserving behavior.
- Minimizes tool schema tokens by removing unused tools and shortening docstrings.
- Reduces state injection tokens via selective `{key?}` usage.
- Compresses large tool results before state storage.
- Applies conversation history summarization to reduce history tokens.
- Right-sizes `max_output_tokens` for agents with predictably short outputs.

## Step-by-Step Instructions

1. **Audit token usage** per source using usage metadata:
   ```python
   for event in runner.run(...):
       if event.usage_metadata:
           input_tokens = event.usage_metadata.prompt_token_count or 0
           output_tokens = event.usage_metadata.candidates_token_count or 0
   ```

2. **Measure instruction token count**:
   ```python
   import google.generativeai as genai
   model = genai.GenerativeModel("gemini-2.0-flash")
   instruction_tokens = model.count_tokens(instruction_text).total_tokens
   ```

3. **Shorten instructions**: Remove redundant phrasing, condense examples.
   Target < 500 tokens for most agents.

4. **Remove unused tools**: Each tool schema = 50–500 tokens. Remove tools not
   needed for the current agent's function.

5. **Shorten tool docstrings**: Keep them to 1 line + parameter descriptions.

6. **Selectively inject state**: Replace always-injected `{key?}` with conditional
   injection in `InstructionProvider`:
   ```python
   def selective_instruction(ctx: ReadonlyContext) -> str:
       step = ctx.state.get("booking_step", "search")
       if step == "confirm":
           return base_instruction + f"\nSelected: {ctx.state.get('selected_flight', '')}"
       return base_instruction
   ```

7. **Compress tool results** before storing in state:
   ```python
   results = search_flights(query)[:5]  # Keep top 5 only
   tool_context.state["search_results"] = compact_serialise(results)
   ```

8. **Enable history summarization** after 20 turns (see `memory-summarization` skill).

9. **Right-size `max_output_tokens`** for short-response agents:
   ```python
   config = GenerateContentConfig(max_output_tokens=256)  # For 1-3 sentence replies
   ```

## Input Format

```python
{
  "operation": "agent_invocation",
  "execution_time_ms": 3200,
  "token_usage": 8500,
  "memory_usage_mb": 64,
  "cost_estimate": 0.0042,
  "retrieval_latency_ms": 0,
  "cache_available": False,
  "parallelizable": False,
  "context_size_tokens": 8500
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "instruction_reduction + tool_pruning + result_compression",
  "performance_improvement": {
    "latency_reduction_ms": 400,
    "token_reduction": 3200,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.0016,
    "throughput_improvement": 1.2
  }
}
```

## Error Handling

- **Instruction too short → agent behavior degrades**: Restore removed sections.
  Apply binary search: remove half the instruction, test quality, restore if needed.
- **Tool removed that agent needed**: Check eval results for `tool_not_found` errors.
  Restore the tool or rename it.
- **`max_output_tokens` too low → response truncated**: Increase in 100-token increments
  until truncation disappears. Test with the longest expected responses.
- **Token count still high after compression** (history dominates): Enable
  history summarization. See `google-adk-memory-summarization`.

## Examples

### Example 1 — Token-audited agent with selective injection

```python
import google.generativeai as genai
from google.adk.agents import LlmAgent
from google.adk.agents.readonly_context import ReadonlyContext
from google.genai.types import GenerateContentConfig

def count_tokens(text: str) -> int:
    """Estimates token count for a text string."""
    model = genai.GenerativeModel("gemini-2.0-flash")
    return model.count_tokens(text).total_tokens

BASE_INSTRUCTION = (
    "You are FlightBot. Help the user book flights efficiently. "
    "Current step: {booking_step?}."
)

def budget_aware_instruction(ctx: ReadonlyContext) -> str:
    """Only injects state values relevant to the current step."""
    step = ctx.state.get("booking_step", "search")
    parts = [BASE_INSTRUCTION.format()]
    if step == "confirm":
        parts.append(f"Selected: {ctx.state.get('selected_flight', 'none')}")
    elif step == "payment":
        parts.append(f"Ref: {ctx.state.get('booking_reference', 'none')}")
    return "\n".join(parts)

TOKEN_EFFICIENT_CONFIG = GenerateContentConfig(
    temperature=0.3,
    max_output_tokens=300,  # Right-sized for chat responses
)

efficient_agent = LlmAgent(
    name="token_efficient_agent",
    model="gemini-2.0-flash",
    instruction=budget_aware_instruction,
    tools=[search_flights, confirm_booking, process_payment],  # Only tools in scope
    generate_content_config=TOKEN_EFFICIENT_CONFIG,
)
```

### Example 2 — Token usage monitor across invocations

```python
import logging
from google.adk.runners import Runner
from google.genai.types import Content, Part

logger = logging.getLogger("token_monitor")

def run_with_token_audit(runner: Runner, user_id: str, session_id: str, query: str) -> dict:
    """Runs the agent and audits token usage per invocation."""
    msg = Content(parts=[Part(text=query)])
    total_input = 0
    total_output = 0
    response_text = ""

    for event in runner.run(user_id=user_id, session_id=session_id, new_message=msg):
        if event.usage_metadata:
            total_input = event.usage_metadata.prompt_token_count or total_input
            total_output += event.usage_metadata.candidates_token_count or 0
        if event.is_final_response() and event.content and event.content.parts:
            response_text = event.content.parts[0].text or ""

    cost_estimate = (total_input / 1_000_000 * 0.075) + (total_output / 1_000_000 * 0.30)
    logger.info(f"Token audit: input={total_input}, output={total_output}, cost=${cost_estimate:.6f}")
    return {
        "response": response_text,
        "input_tokens": total_input,
        "output_tokens": total_output,
        "estimated_cost_usd": round(cost_estimate, 6),
    }
```

## Edge Cases

- **Instruction dynamic expansion** (`InstructionProvider` adds many tokens for
  some users): Cap injected values at 200 chars per field. Log warning when
  injected instruction exceeds 1000 tokens.
- **Tool with very long schema** (complex parameter types): Break into multiple
  simpler tools or use a single `action` parameter with enum values.
- **History token count accelerates** (tool results injected into history): Truncate
  tool results to 500 chars before returning to the agent. Long tool results
  are included verbatim in session history.

## Integration Notes

- **`google-adk-memory-summarization`**: Summarizes history to reduce history tokens.
- **`google-adk-context-optimization`**: Broader context management strategies.
- **`google-adk-cost-optimization`**: Token reduction is the primary cost lever.
- **`google-adk-caching-strategies`**: Avoid recomputing results (saves tool execution
  tokens in multi-turn reasoning loops).
- **`google-adk-prompt-optimization`**: Prompt engineering to reduce instruction length
  while preserving quality.
