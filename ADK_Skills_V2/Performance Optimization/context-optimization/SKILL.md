---
name: context-optimization
description: >
  Use this skill when optimizing the content and size of the LLM context window
  for Google ADK LlmAgent to maximize reasoning quality while minimizing token
  waste. Covers selective state injection, dynamic instruction construction,
  context pruning, relevance scoring of injected content, and context window
  budget allocation across instruction, history, and tool schemas.
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

This skill optimizes the content and structure of the LLM context window for Google
ADK `LlmAgent` to maximize the signal-to-noise ratio of the injected context.
The context window contains: system instruction, injected state values, tool schemas,
conversation history, and the user message. Poor context optimization leads to:
noise diluting reasoning quality, redundant content wasting token budget, irrelevant
history inflating input cost, and important content being "buried" in a large context.
This skill applies targeted content selection and prioritization to extract maximum
reasoning value from every token in the context window budget.

## When to Use

- When the agent gives incorrect or inconsistent answers despite having access
  to the right data (likely buried in noisy context).
- When context window utilization is > 60% of budget and injection is growing.
- When the instruction injects the same state values regardless of the current turn.
- When tool schemas of unused tools are consuming context budget.
- When multiple `{key?}` injections deliver redundant or overlapping information.

## When NOT to Use

- Do not remove context that the agent genuinely needs for correct reasoning —
  validate quality after every context reduction.
- Do not apply context optimization without a baseline quality eval to measure
  against.
- Do not reduce conversation history below the last 5 turns in interactive chat agents.

## Google ADK Context

- **`{key?}` injection**: State values injected into the `LlmAgent.instruction`.
  Each injection adds tokens. `{key?}` resolves to empty string if the key doesn't exist.
- **`InstructionProvider`**: A callable that returns the instruction string.
  Enables dynamic, per-turn context construction.
- **`before_agent_callback`**: Pre-computes the content to inject for the current
  turn and stores it in `temp:` keys for `{key?}` injection.
- **Tool schema tokens**: Each tool in `LlmAgent.tools` adds its complete JSON schema
  to the context on every request. Remove tools not needed for the current step.
- **Context window budget allocation**:
  - Instruction: ≤ 20% of context window
  - Injected state: ≤ 15%
  - Tool schemas: ≤ 5%
  - History: ≤ 50%
  - User message: remaining
- **`temp:` prefix**: Auto-cleared after each turn. Perfect for turn-specific
  context computed in `before_agent_callback`.
- **Relevance scoring**: Score each potential injection by relevance to the current
  turn. Inject only those scoring above a threshold.

## Capabilities

- Constructs dynamic, step-adaptive instructions via `InstructionProvider`.
- Selectively injects only turn-relevant state values.
- Computes and enforces context window token budget allocation.
- Removes unused tool schemas to reduce context overhead.
- Pre-computes turn-specific context in `before_agent_callback`.
- Scores injection relevance by current workflow step.

## Step-by-Step Instructions

1. **Measure current context window utilization**:
   ```python
   for event in runner.run(...):
       if event.usage_metadata:
           used = event.usage_metadata.prompt_token_count or 0
           utilization = used / CONTEXT_WINDOW
   ```

2. **Audit injection content** — list all `{key?}` references in the instruction:
   - Check which are used on every turn vs. only some turns.

3. **Convert static instruction to `InstructionProvider`**:
   ```python
   def smart_instruction(ctx: ReadonlyContext) -> str:
       step = ctx.state.get("booking_step", "search")
       if step == "search":
           return BASE_INSTRUCTION  # Minimal injection for search step
       elif step == "confirm":
           return BASE_INSTRUCTION + f"\nFlight: {ctx.state.get('selected_flight', '')}"
   ```

4. **Remove unused tools** from the agent's tool list for each workflow step.
   (Or use a `LoopAgent` with different tool sets per step.)

5. **Compress injected values** before storing to reduce injection token count:
   ```python
   compact_results = [{"id": r["id"], "price": r["price"]} for r in full_results]
   tool_context.state["search_results_compact"] = compact_results
   ```

6. **Apply relevance-based injection**: Before each turn, score each potential
   injection by relevance to the current query. Include only top-3.

7. **Summarize history** when history tokens exceed 40% of context window.

## Input Format

```python
{
  "operation": "instruction_optimization",
  "execution_time_ms": 3200,
  "token_usage": 15000,
  "memory_usage_mb": 64,
  "cost_estimate": 0.0075,
  "retrieval_latency_ms": 0,
  "cache_available": False,
  "parallelizable": False,
  "context_size_tokens": 15000
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "selective_injection + tool_pruning + instruction_compaction",
  "performance_improvement": {
    "latency_reduction_ms": 600,
    "token_reduction": 5500,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.00275,
    "throughput_improvement": 1.3
  }
}
```

## Error Handling

- **Agent misses required context** after injection reduction: Restore the removed
  injection. Binary search over injections to find the minimal required set.
- **`temp:` computed context is stale** (computed in previous turn's callback):
  `temp:` keys auto-clear. Verify `before_agent_callback` runs before each turn.
- **Too many tool removes break agent capability**: Only remove tools that are
  never called at the current workflow step. Verify with integration tests.
- **Instruction compaction produces worse answers**: The instruction was not
  redundant — restore it. Quality eval is mandatory after every compaction.

## Examples

### Example 1 — Step-adaptive instruction provider with minimal injection

```python
from google.adk.agents.readonly_context import ReadonlyContext

STEP_INSTRUCTIONS = {
    "search": (
        "You are FlightBot. Help the user search for flights. "
        "Use search_flights to find options."
    ),
    "confirm": (
        "You are FlightBot. Confirm the booking details below and ask for user approval.\n"
        "Selected flight: {selected_flight?}\n"
        "Price: {selected_price?}"
    ),
    "payment": (
        "You are FlightBot. Process payment for booking {booking_reference?}. "
        "Call process_payment when the user confirms."
    ),
    "complete": (
        "You are FlightBot. Booking {booking_reference?} is confirmed. "
        "Provide a confirmation summary."
    ),
}

def adaptive_instruction(ctx: ReadonlyContext) -> str:
    """Returns a minimal, step-specific instruction — no wasted tokens."""
    step = ctx.state.get("booking_step", "search")
    template = STEP_INSTRUCTIONS.get(step, STEP_INSTRUCTIONS["search"])
    # Template already has only the {key?} values needed for this step
    return template
```

### Example 2 — Context budget monitor and trimmer

```python
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content

CONTEXT_WINDOW = 1_000_000
MAX_INSTRUCTION_PCT = 0.20
MAX_STATE_INJECTION_PCT = 0.15

def context_budget_enforcer(callback_context: CallbackContext) -> Optional[Content]:
    """Enforces context window budget allocation per turn."""
    tokens_used = int(callback_context.state.get("session_total_tokens", 0))
    history_pct = tokens_used / CONTEXT_WINDOW

    # If history alone uses > 60% of context window, trigger summarization signal
    if history_pct > 0.60:
        callback_context.state["needs_summarization"] = True

    # Trim oversized state values that might be injected
    for key, val in list(callback_context.state.items()):
        if key.startswith("temp:") or key.startswith("cache:"):
            if len(str(val)) > 3000:
                callback_context.state[key] = str(val)[:3000] + "...[trimmed]"

    return None
```

## Edge Cases

- **Instruction template with undefined `{key?}`**: ADK's `{key?}` resolves to
  empty string for missing keys — no error, but the absent injection may be confusing.
  Log a warning when a critical `{key?}` resolves to empty.
- **Context optimization improves quality** (not just tokens): Removing irrelevant
  context reduces noise. The agent's reasoning improves when the context is signal-rich.
- **Multi-agent pipeline context** (each sub-agent has its own context): Optimize
  each sub-agent's context independently. A `ParallelAgent` sub-agent should have
  minimal context — only its specific task data.

## Integration Notes

- **`google-adk-token-usage-optimization`**: Context optimization is the content-level
  lever; token optimization is the count-level lever.
- **`google-adk-working-memory`**: Context optimization directly manages what
  enters working memory each turn.
- **`google-adk-memory-summarization`**: Summarizes history to free context window space.
- **`google-adk-context-window-management`**: Monitors the budget. Context
  optimization is the strategy; budget management is the enforcement layer.
