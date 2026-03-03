---
name: working-memory
description: >
  Use this skill when a Google ADK agent needs to manage the active, in-context
  data available to the LLM during the current invocation — the contents of the
  context window assembled from session state, instruction, tool results, and
  conversation history. Covers strategies for populating, pruning, and prioritizing
  working memory to maximize reasoning effectiveness within the context window.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: memory-management
---

## Purpose

This skill manages working memory for Google ADK agents — the active set of
information present in the LLM's context window during a specific invocation.
Working memory in ADK comprises: the system instruction, injected state values,
the conversation history (session events), and tool results. This skill covers
strategies for deciding what to include in working memory, how to prioritize
the most relevant data, how to prune stale or irrelevant information, and how
to structure context injection for maximum reasoning effectiveness.

## When to Use

- When the agent's context window is filling up and context management is needed.
- When different turns in a conversation require different active context subsets.
- When deciding which tool results, state values, and history events to include
  in the instruction for the current turn.
- When implementing `before_agent_callback` to populate context-window-relevant
  data before the LLM call.
- When the agent's reasoning quality is degraded by irrelevant context cluttering
  the context window.

## When NOT to Use

- Do not conflate working memory management with session state management — they
  are different layers. Working memory is about what is in the context window
  for this invocation; session state is about what is persisted.
- Do not include all available state in working memory — inject only what is
  relevant to the current turn.
- Do not manually manage conversation history length unless approaching the
  model's context window limit.

## Google ADK Context

- **Context window composition**: `[system_instruction] + [injected_state_via_{key?}] + [tool_schemas] + [conversation_history_events] + [latest_user_message]`
- **`{key?}` injection**: Selectively injects session state values into the
  instruction. Each injection adds tokens — only inject relevant values.
- **`before_agent_callback`**: The hook for pre-populating working memory.
  Compute and store in state the exact context the current turn needs, then
  inject via `{key?}`.
- **`after_agent_callback`**: The hook for cleaning up transient working memory
  (removing context that was only needed for this turn).
- **`temp:` prefix**: `temp:` state keys are cleared at the end of each invocation.
  Use for working memory that is needed only for the current turn but should not
  persist to the next.
- **ADK session events**: The model sees all events in the session. As sessions
  grow, conversation history occupies more of the context window.
- **Working memory budget**: `available_for_content = context_window - instruction_tokens - tool_schema_tokens`

## Capabilities

- Selectively injects relevant state into the LLM context for the current turn.
- Manages context window budget across instruction, history, and tool results.
- Uses `temp:` keys for single-turn working context that auto-clears.
- Populates turn-specific context in `before_agent_callback`.
- Clears stale working context in `after_agent_callback`.
- Prioritizes high-relevance data over low-relevance data in context construction.

## Step-by-Step Instructions

1. **Audit the context window composition** for the current agent:
   - Measure instruction tokens, tool schema tokens, history tokens.
   - Compute the available budget for injected state content.

2. **Identify what working memory is needed for this turn** based on intent:
   - Step-specific: inject only data relevant to the current workflow step.
   - Query-specific: inject only the search results relevant to the current query.

3. **Populate working memory** in `before_agent_callback`:
   ```python
   def populate_working_memory(callback_context: CallbackContext):
       step = callback_context.state.get("booking_step", "search")
       if step == "confirm":
           callback_context.state["temp:active_context"] = (
               f"Selected: {callback_context.state.get('selected_flight', 'none')}"
           )
       return None
   ```

4. **Inject via instruction** using `{temp:active_context?}`:
   ```python
   instruction = "Current context: {temp:active_context?}\nHelp the user proceed."
   ```

5. **Prune irrelevant state injections** by using `{key?}` only for keys that
   are non-null and relevant for the current turn.

6. **After the turn**, clean up non-`temp:` working memory that should not persist:
   ```python
   def cleanup_working_context(callback_context: CallbackContext, response):
       for key in ["active_search_results", "temp_scoring_data"]:
           callback_context.state.pop(key, None)
       return None
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "working",
  "content": "current_step=confirm|selected_flight=BA178",
  "metadata": {"turn": 4, "context_tokens_used": 8200},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "temp:active_context:turn_4",
  "memory_retrieved": [{"key": "temp:active_context", "value": "Selected: BA178"}],
  "memory_injected": True
}
```

## Error Handling

- **Context window exceeded**: Check token budget before each injection.
  Reduce history window or apply context summarization.
- **`temp:` key not cleared** (custom session backend): Verify your session
  backend clears `temp:` keys at invocation end. Implement manually if needed.
- **Working memory injects stale data**: Validate that `temp:` values are set
  freshly each turn in `before_agent_callback`.
- **Too many `{key?}` injections inflating instruction**: Count tokens in the
  assembled instruction. Cap at 50% of the context window for the instruction.

## Examples

### Example 1 — Step-adaptive working memory population

```python
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content

def populate_step_context(callback_context: CallbackContext) -> Optional[Content]:
    """Populates turn-specific working memory based on the current booking step."""
    step = callback_context.state.get("booking_step", "search")
    context_parts = []

    if step == "search":
        last_query = callback_context.state.get("last_search_query", "")
        if last_query:
            context_parts.append(f"Last search: {last_query}")

    elif step == "confirm":
        flight = callback_context.state.get("selected_flight", "")
        price = callback_context.state.get("selected_price", "")
        context_parts.append(f"Selected flight: {flight} at ${price}")

    elif step == "payment":
        ref = callback_context.state.get("booking_reference", "")
        context_parts.append(f"Booking reference: {ref} — pending payment")

    callback_context.state["temp:step_context"] = (
        " | ".join(context_parts) if context_parts else "(no active context)"
    )
    return None
```

### Example 2 — Working memory budget calculator

```python
def estimate_available_tokens(
    instruction_tokens: int,
    tool_schemas_tokens: int,
    history_tokens: int,
    context_window: int = 1_000_000
) -> dict:
    """Estimates token budget remaining for content injection."""
    used = instruction_tokens + tool_schemas_tokens + history_tokens
    available = context_window - used
    return {
        "total_context_window": context_window,
        "tokens_used": used,
        "tokens_available_for_injection": max(0, available),
        "utilization_pct": round(used / context_window * 100, 2),
    }
```

## Edge Cases

- **Long tool results expand working memory unexpectedly**: Truncate tool result
  text before storing in state. Keep tool responses < 2000 tokens.
- **History grows across many turns**: As turns accumulate, the conversation
  history occupies more context. Apply context summarization after N turns.
- **Working memory and structured output schema conflict**: The schema tokens
  consume context window budget. Factor schema token count into budget calculations.

## Integration Notes

- **`google-adk-context-window-management`**: Full context window management
  strategies built on working memory principles.
- **`google-adk-short-term-memory`**: The persistent layer below working memory.
  Short-term state is the source from which working memory is populated.
- **`google-adk-memory-summarization`**: When working memory (history) grows too
  large, summarization compresses it to fit the context window.
- **`google-adk-context-injection`**: The `{key?}` mechanism for injecting working
  memory content into the LLM's context.
