---
name: context-window-management
description: >
  Use this skill when managing the total token budget in the LLM context window
  for a Google ADK LlmAgent. Covers measuring context window utilization,
  prioritizing which content to include, implementing sliding window history,
  applying summarization triggers, and preventing context window overflow in
  long-running multi-turn ADK agent sessions.
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

This skill manages the LLM context window for Google ADK `LlmAgent` — ensuring
the total tokens consumed by the system instruction, injected state, tool schemas,
and conversation history don't exceed the model's context window limit. For long-running
multi-turn sessions, conversation history grows and can approach the 1M token
limit (Gemini Flash) or 2M token limit (Gemini Pro). This skill provides monitoring,
prioritization, sliding window, and summarization strategies to maintain optimal
context window utilization throughout the agent's lifetime.

## When to Use

- When the session has accumulated > 20 turns and context window management
  is needed proactively.
- When `event.usage_metadata.prompt_token_count` approaches 80% of the model's
  context window.
- When designing a long-lived agent (customer support, complex workflow orchestration)
  where sessions regularly exceed 50 turns.
- When implementing multi-agent pipelines where each sub-agent's context adds to
  the total token budget.
- When the agent raises a context window overflow error.

## When NOT to Use

- Do not apply context window management for short sessions (< 20 turns) — it
  adds unnecessary complexity.
- Do not truncate conversation history without summarizing it first — raw truncation
  causes the agent to lose important earlier context.
- Do not monitor context window by counting characters — use actual token counts
  from `event.usage_metadata.prompt_token_count`.

## Google ADK Context

- **`event.usage_metadata.prompt_token_count`**: The actual input token count
  for each LLM invocation. Monitor this across turns to track utilization growth.
- **Context window sizes**: Flash = 1M tokens, Flash-Lite = 1M tokens, Pro = 2M tokens.
- **Context window composition**:
  `instruction_tokens + injected_state_tokens + tool_schema_tokens + history_tokens + latest_message_tokens`
- **`before_agent_callback`**: The monitoring and management hook. Check token
  budget before each invocation and apply management strategies.
- **Sliding window**: Keep only the last N events in the active context. Older
  events are summarized into `session_summary`.
- **Summarization trigger**: When `prompt_token_count > THRESHOLD`, trigger
  `auto_summarize_callback` to compress old history.
- **ADK session events**: All `session.events` are visible to the model. Managing
  the event count directly manages the history token count.
- **Tool schema tokens**: Each tool adds its schema to the context on every call.
  Minimize the number of tools per agent.

## Capabilities

- Monitors context window utilization per invocation.
- Detects warning and critical thresholds.
- Implements sliding window history management.
- Triggers automatic summarization at configurable thresholds.
- Reduces injected state tokens by selective injection.
- Provides context budget estimates for planning.
- Logs context window metrics for observability.

## Step-by-Step Instructions

1. **Define context window limits** for the deployed model:
   ```python
   CONTEXT_WINDOWS = {
       "gemini-2.0-flash": 1_000_000,
       "gemini-2.0-flash-lite": 1_000_000,
       "gemini-2.0-pro": 2_000_000,
   }
   WARN_THRESHOLD = 0.75   # 75% → warn
   CRITICAL_THRESHOLD = 0.90  # 90% → trigger summarization
   ```

2. **Monitor token count** from response event metadata:
   ```python
   for event in runner.run(...):
       if event.usage_metadata:
           input_tokens = event.usage_metadata.prompt_token_count or 0
           tool_context.state["session_total_tokens"] = input_tokens
   ```

3. **Check budget in `before_agent_callback`** and trigger management:
   ```python
   def context_budget_guard(callback_context: CallbackContext):
       tokens_used = int(callback_context.state.get("session_total_tokens", 0))
       context_window = CONTEXT_WINDOWS.get(callback_context.state.get("model", "gemini-2.0-flash"), 1_000_000)
       pct = tokens_used / context_window
       if pct >= CRITICAL_THRESHOLD:
           callback_context.state["needs_summarization"] = True
       elif pct >= WARN_THRESHOLD:
           callback_context.state["context_warning"] = True
       return None
   ```

4. **Reduce injected state** by using conditional `{key?}` injection:
   - Only inject `{user:name?}` when name is known.
   - Remove stale `{search_results?}` from injection when results are no longer relevant.

5. **Implement sliding window**: When event count > MAX_HISTORY_EVENTS, summarize
   older events and remove them from the instruction-visible history.

6. **Minimize tool schema tokens**: Remove unused tools from the agent's tool list.
   Each tool's JSON schema is included in every LLM request.

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "working",
  "content": "context_window_check at turn 35",
  "metadata": {"model": "gemini-2.0-flash", "tokens_used": 780000, "context_window": 1000000},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": False,
  "memory_id": None,
  "memory_retrieved": [
    {
      "tokens_used": 780000, "context_window": 1000000,
      "utilization_pct": 78.0, "status": "warning",
      "action": "summarization_recommended"
    }
  ],
  "memory_injected": False
}
```

## Error Handling

- **Context window overflow** (`INVALID_ARGUMENT: context window exceeded`):
  Immediately apply emergency summarization. As a fallback, drop the oldest half
  of session events and continue.
- **`prompt_token_count` always 0** (`usage_metadata` not populated): ADK only
  populates usage metadata on LLM response events — filter for those events.
- **Instruction token count exceeds 30% of context window**: Reduce `{key?}`
  injections. Move large injected content to tool return values instead.
- **Tool schema tokens inflate context** (many tools): Remove low-use tools.
  Group related functionality into a single tool with an `action` parameter.

## Examples

### Example 1 — Full context window guardian callback

```python
import time
import logging
import google.generativeai as genai
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content, Part

logger = logging.getLogger("context_window")

CONTEXT_WINDOWS = {"gemini-2.0-flash": 1_000_000, "gemini-2.0-pro": 2_000_000}
WARN_THRESHOLD = 0.75
CRITICAL_THRESHOLD = 0.90
MAX_HISTORY_EVENTS = 40
SUMMARIZE_COOLDOWN = 300

def context_window_guardian(callback_context: CallbackContext) -> Optional[Content]:
    """Guards against context window overflow and triggers management strategies."""
    model = callback_context.state.get("app:model", "gemini-2.0-flash")
    ctx_window = CONTEXT_WINDOWS.get(model, 1_000_000)
    tokens_used = int(callback_context.state.get("session_total_tokens", 0))
    pct = tokens_used / ctx_window if tokens_used else 0.0

    logger.info(f"Context window: {tokens_used}/{ctx_window} ({pct:.1%})")

    # Critical: trigger emergency summarization
    if pct >= CRITICAL_THRESHOLD:
        last_summarized = float(callback_context.state.get("summary_updated_at", 0))
        if time.time() - last_summarized > SUMMARIZE_COOLDOWN:
            events = callback_context._invocation_context.session.events
            older_events = events[:-10]  # Summarize all but last 10
            if older_events:
                history_lines = []
                for e in older_events[:30]:
                    if e.content and e.content.parts:
                        role = "User" if e.author == "user" else "Agent"
                        text = " ".join(p.text for p in e.content.parts if hasattr(p, "text") and p.text)
                        if text:
                            history_lines.append(f"{role}: {text[:200]}")
                if history_lines:
                    try:
                        resp = genai.generate_content(
                            model="gemini-2.0-flash-lite",
                            contents=[{"parts": [{"text": (
                                "Summarize in ≤100 words. Preserve: goals, actions, preferences:\n\n"
                                + "\n".join(history_lines)
                            )}]}]
                        )
                        summary = (resp.text or "").strip()[:1000]
                        callback_context.state["session_summary"] = summary
                        callback_context.state["summary_updated_at"] = time.time()
                        callback_context.state["session_total_tokens"] = 0  # Reset estimate
                        logger.warning(f"Emergency summarization applied. Session: {callback_context.session_id}")
                    except Exception as exc:
                        logger.error(f"Summarization failed: {exc}")

    # Warning: log and flag
    elif pct >= WARN_THRESHOLD:
        callback_context.state["context_warning"] = True
        logger.warning(f"Context window at {pct:.1%}. Compression recommended.")

    return None
```

### Example 2 — Token count tracking from event stream

```python
from google.adk.runners import Runner
from google.genai.types import Content, Part

async def run_with_token_tracking(runner: Runner, user_id: str, session_id: str,
                                   query: str, session_service) -> str:
    """Runs the agent and tracks cumulative context window usage."""
    msg = Content(parts=[Part(text=query)])
    response_text = ""
    for event in runner.run(user_id=user_id, session_id=session_id, new_message=msg):
        if event.usage_metadata and event.usage_metadata.prompt_token_count:
            # Update cumulative token count in state via next callback
            pass  # In practice, write to state in a callback
        if event.is_final_response() and event.content and event.content.parts:
            response_text = event.content.parts[0].text or ""
    return response_text
```

## Edge Cases

- **Tool schema tokens dominate context budget**: Profile with 1 tool vs. 10 tools.
  Consider splitting the agent into specialized sub-agents with fewer tools each.
- **Single very long user message** exhausts context budget: Add a maximum input
  length check in `before_agent_callback`. Return an error if input > N chars.
- **Model upgraded with different context window**: Store the model name in
  `app:model` state for `CONTEXT_WINDOWS` lookup. Update when the model changes.
- **Summarization doesn't reduce token count** (old events still in session.events):
  ADK includes all `session.events` in the context. To truly reduce history,
  implement a custom session service that prunes old events on commit.

## Integration Notes

- **`google-adk-memory-summarization`**: The primary strategy for freeing context
  window space. Triggered by the context budget guardian in this skill.
- **`google-adk-working-memory`**: Defines the composition of context window
  content. This skill manages its total size.
- **`google-adk-token-limits`**: Model-level `max_output_tokens` reduces output
  token consumption. This skill manages the input/history side.
- **`google-adk-memory-compression`**: Compresses individual state values to
  reduce injection token overhead.
