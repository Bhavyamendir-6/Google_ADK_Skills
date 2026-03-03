---
name: google-adk-context-summarization
description: >
  Use this skill when a Google ADK agent needs to compress long conversation
  history into a concise summary stored in session.state, enabling efficient
  context management without losing essential information. Covers LLM-based
  summarization of session.events, storing the summary in state, and injecting
  the summary into subsequent agent instructions.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: state-and-context-handling
---

## Purpose

This skill compresses accumulated `session.events` into a concise textual
summary stored in `session.state["conversation_summary"]`. The summary is then
injected into subsequent agent instructions, allowing the LLM to maintain
conversational continuity without requiring the full event history in context.
This is applied when context pruning alone would lose semantically important
information.

## When to Use

- When `session.events` exceeds a turn threshold (e.g., 30 events) and pruning
  alone loses critical context.
- Before a context window reset where the full history cannot be retained.
- When implementing memory-efficient long-horizon conversational agents.
- When transitioning between conversation phases (e.g., onboarding → task).

## When NOT to Use

- Do not summarize during every turn — only when the event count crosses a
  threshold (e.g., every 20 new events).
- Do not use for short conversations (< 10 turns) — the overhead is not worth it.
- Do not summarize `temp:` or computation-only events — only user/agent
  message events carry semantic value for summarization.

## Google ADK Context

- **`session.events`**: The event history to be summarized. Filter to include
  only user messages and agent final responses for summarization input.
- **`session.state["conversation_summary"]`**: The designated state key for
  storing the generated summary. Inject via `{conversation_summary?}` in
  `LlmAgent.instruction`.
- **`tool_context.state`**: Used in summarization tool functions to write the
  summary back to state.
- **ADK Execution Lifecycle**: Summarization occurs as a tool call (triggered
  by the agent) or within `before_agent_callback` (automatic, periodic).
- **`LlmAgent` with summarization**: Use a dedicated lightweight `LlmAgent`
  (e.g., with `gemini-2.0-flash`) as the summarizer to avoid polluting the
  main agent's context.

## Capabilities

- Generates LLM-based conversation summaries from `session.events`.
- Stores the summary in `session.state["conversation_summary"]`.
- Injects the summary into main agent instructions via `{conversation_summary?}`.
- Implements periodic summarization triggered by event count threshold.
- Replaces the full event history with the summary + recent events only.
- Uses a dedicated lightweight summarizer agent to minimize cost.

## Step-by-Step Instructions

1. **Determine the summarization threshold**: Default is every 20 events.
   Check `len(session.events)` against the threshold.

2. **Extract summarizable events**: Filter `session.events` to keep only
   user messages and agent final responses (drop tool call/result events).

3. **Build the summarization prompt**:
   ```python
   history_text = "\n".join(
       f"{e.author}: {e.content.parts[0].text}"
       for e in events_to_summarize
       if e.content and e.content.parts
   )
   prompt = f"Summarize this conversation concisely (max 200 words):\n{history_text}"
   ```

4. **Call a lightweight LLM** to generate the summary. Use a separate model
   call (not the main agent) to avoid polluting the conversation history.

5. **Store the summary in `tool_context.state`**:
   ```python
   tool_context.state["conversation_summary"] = summary_text
   tool_context.state["summary_as_of_event"] = len(session.events)
   ```

6. **Prune the summarized events** from `session.events`, keeping only the
   post-summary events (the `google-adk-context-pruning` skill handles this).

7. **Inject the summary into the main agent instruction**:
   ```python
   LlmAgent(instruction="Previous context: {conversation_summary?}\n\nNow assist the user.")
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_xyz",
  "state": {
    "conversation_summary": "",          # Will be populated by this skill
    "summary_as_of_event": 0
  },
  "metadata": {
    "summarization_threshold": 20,       # Events count before triggering
    "max_summary_words": 200,
    "summarizer_model": "gemini-2.0-flash"
  }
}
```

## Output Format

```python
{
  "state_updated": True,
  "state_key": "conversation_summary",
  "state_value": "The user asked about booking a flight from NYC to London. The agent confirmed availability for March 15th and the user selected economy class. Payment details are pending.",
  "summary_as_of_event": 24,
  "events_summarized": 20
}
```

## Error Handling

- **LLM summarization call fails**: Fall back to a simple concatenation of the
  last 5 user messages as the summary. Store with a `[FALLBACK]` prefix.
- **`conversation_summary` key missing**: Default to empty string. The agent
  instruction `{conversation_summary?}` handles missing keys gracefully.
- **Summary too long**: Truncate to `max_summary_words` words after generation.
- **Summarization triggered on empty events**: Check `len(events_to_summarize) > 0`
  before calling the LLM.
- **Concurrent summarization in parallel agents**: Use `temp:summarization_in_progress`
  as a mutex flag in state to prevent duplicate summarization.

## Examples

### Example 1 — Summarization tool function

```python
import asyncio
from google.adk.tools import ToolContext
from google.genai import Client

genai_client = Client()

async def summarize_conversation(tool_context: ToolContext) -> dict:
    """Summarizes the current session's conversation history and stores it in state.

    Args:
        tool_context (ToolContext): ADK tool context with access to session events.

    Returns:
        dict: Contains 'summary' (str) and 'events_summarized' (int).
    """
    session = tool_context._invocation_context.session
    events = session.events

    # Filter to semantic events only
    semantic_events = [
        e for e in events
        if e.content and e.content.parts and e.author in ("user", "agent")
    ]

    if len(semantic_events) < 5:
        return {"summary": "", "events_summarized": 0}

    history_text = "\n".join(
        f"{e.author.upper()}: {e.content.parts[0].text}"
        for e in semantic_events
        if hasattr(e.content.parts[0], "text")
    )

    response = genai_client.models.generate_content(
        model="gemini-2.0-flash",
        contents=f"Summarize this conversation in max 150 words:\n\n{history_text}",
    )
    summary = response.text.strip()

    tool_context.state["conversation_summary"] = summary
    tool_context.state["summary_as_of_event"] = len(events)

    return {"summary": summary, "events_summarized": len(semantic_events)}
```

### Example 2 — Periodic summarization in before_agent_callback

```python
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content

SUMMARIZATION_THRESHOLD = 20

def maybe_summarize(callback_context: CallbackContext) -> Optional[Content]:
    """Triggers summarization when event count crosses threshold."""
    session = callback_context._invocation_context.session
    last_summary_event = callback_context.state.get("summary_as_of_event", 0)
    new_events_since_summary = len(session.events) - last_summary_event

    if new_events_since_summary >= SUMMARIZATION_THRESHOLD:
        # Trigger summarization synchronously (simplified)
        import asyncio
        asyncio.create_task(summarize_and_prune(callback_context))

    return None
```

### Example 3 — Main agent with summary injection

```python
from google.adk.agents import LlmAgent

main_agent = LlmAgent(
    name="main_assistant",
    model="gemini-2.0-flash",
    instruction=(
        "You are a helpful assistant.\n"
        "Conversation summary so far: {conversation_summary?}\n\n"
        "Continue assisting the user from where we left off."
    ),
    tools=[summarize_conversation],
)
```

## Edge Cases

- **First session turn (no events)**: Skip summarization. `summary_as_of_event`
  defaults to 0 and `conversation_summary` defaults to `""`.
- **All events are tool events** (no user/agent messages): Summarization of
  tool events alone is not meaningful. Skip if no semantic events exist.
- **Summary grows across repeated summarizations**: On each trigger, summarize
  the previous summary + new events together to produce a cumulative summary.
- **Multi-language conversations**: The LLM summarizer will naturally produce
  the summary in the dominant language. Store the language code in state too.

## Integration Notes

- **`google-adk-context-pruning`**: Apply AFTER summarization. Prune old events
  once the summary is stored in state.
- **`google-adk-context-injection`**: Injects `{conversation_summary?}` into
  the main agent instruction automatically.
- **`google-adk-checkpointing`**: Checkpoint the session state (including the
  summary) after each summarization event.
- **`google-adk-cross-session-memory`**: Store the final conversation summary
  in `user:` scoped state for retrieval in future sessions.
