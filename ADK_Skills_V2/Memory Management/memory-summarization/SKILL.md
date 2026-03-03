---
name: memory-summarization
description: >
  Use this skill when a Google ADK agent needs to compress growing conversation
  history, episode logs, or session narratives into compact summaries using an
  LLM. Covers triggering summarization at N-turn thresholds, implementing a
  summarization agent or tool, storing summaries in user: or session state,
  and replacing full history with summaries for context window management.
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

This skill implements LLM-based memory summarization for Google ADK agents —
compressing long conversation histories, episode logs, and session narratives
into compact, semantically rich summaries that preserve key information within
a token-efficient footprint. As multi-turn sessions grow, the full conversation
history can approach the model's context window limit. Summarization replaces
older turns with a compact summary, freeing context window capacity for new
turns while retaining essential context. This skill covers trigger detection,
summarization agent design, and summary integration into the session state.

## When to Use

- When conversation history in a session exceeds N turns (e.g., > 20 turns).
- When the `session_total_tokens` counter approaches 80% of the context window.
- When the episode log exceeds the working memory budget for context injection.
- When the agent starts a new session and needs to inject a summary of the
  previous session's key outcomes.
- When building a "memory handoff" between sessions for long-running workflows.

## When NOT to Use

- Do not summarize within the first 10 turns — LLM overhead is not justified.
- Do not summarize tool results containing exact values (amounts, codes, dates) —
  exact values are lossy in LLM summaries. Store these separately.
- Do not apply summarization to structured JSON memories — use structural
  compaction instead.

## Google ADK Context

- **Trigger condition**: Check `len(session.events)` or `session_total_tokens`
  in `before_agent_callback` to determine when to trigger summarization.
- **Summarization tool**: A dedicated tool function that calls a separate
  `LlmAgent` or direct Gemini API to summarize the conversation history.
- **`user:session_summary_N`**: Store completed session summaries keyed by
  session number for long-term cross-session memory.
- **`user:ongoing_summary`**: Rolling summary of the current session, updated
  every N turns.
- **`tool_context.state["session_summary"]`**: Session-scoped summary for use
  in the current session.
- **ADK session events**: The full conversation history is in `session.events`.
  Summarize the oldest M events; retain the most recent K events verbatim.
- **Summarization `LlmAgent`**: A no-tools `LlmAgent` with
  `output_schema=SessionSummary` for structured summary output.

## Capabilities

- Detects summarization trigger conditions (turn count, token budget).
- Summarizes conversation history using a dedicated gemini-flash agent.
- Stores rolling summaries in session and user-scoped state.
- Integrates summaries into subsequent agent instructions.
- Preserves key facts (bookings, preferences, decisions) through summarization.
- Implements cross-session summary handoffs via `user:` state.

## Step-by-Step Instructions

1. **Detect trigger** in `before_agent_callback`:
   ```python
   events = callback_context._invocation_context.session.events
   if len(events) > 20 and not callback_context.state.get("summary_updated_at"):
       callback_context.state["needs_summarization"] = True
   ```

2. **Collect text to summarize** from events:
   ```python
   def collect_history_text(events: list, max_events: int = 20) -> str:
       lines = []
       for e in events[:max_events]:
           if e.content and e.content.parts:
               role = "User" if e.author == "user" else "Agent"
               text = " ".join(p.text for p in e.content.parts if hasattr(p, "text") and p.text)
               lines.append(f"{role}: {text[:200]}")
       return "\n".join(lines)
   ```

3. **Call summarization** (direct Gemini API for low latency):
   ```python
   import google.generativeai as genai
   def summarize_history(history_text: str, tool_context: ToolContext) -> dict:
       prompt = (
           "Summarize this conversation in ≤150 words. Preserve: "
           "user goals, key decisions, booked items, preferences, pending actions.\n\n"
           + history_text
       )
       response = genai.generate_content(
           model="gemini-2.0-flash-lite", contents=[{"parts": [{"text": prompt}]}]
       )
       summary = response.text.strip() if response and response.text else "[No summary]"
       tool_context.state["session_summary"] = summary
       tool_context.state["user:last_session_summary"] = summary
       return {"summary_stored": True, "summary_length": len(summary)}
   ```

4. **Inject summary into next instruction**:
   ```python
   instruction = "CONTEXT SUMMARY:\n{session_summary?}\n\nHelp the user continue their task."
   ```

5. **Archive across sessions** by copying to `user:session_summary_N`:
   ```python
   n = int(tool_context.state.get("user:session_count", 0))
   tool_context.state[f"user:session_summary_{n}"] = summary
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "short_term",
  "content": "[full conversation history — 25 turns]",
  "metadata": {"trigger": "turn_count_exceeded", "turns_to_summarize": 20},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "sess_abc123:session_summary",
  "memory_retrieved": [{"content": "User searched JFK→LHR flights. Preferred BA178 at $520. Awaiting payment confirmation.", "type": "summary"}],
  "memory_injected": True
}
```

## Error Handling

- **Summarization loses critical booking reference**: Extract exact values
  (references, amounts) before summarization and re-add them to the summary
  as a structured suffix: `"\n[BOOKING: BOOK-92834]"`.
- **Summarization API call fails**: Catch the error and proceed without summary.
  Log the failure for monitoring.
- **Summary too long** (LLM ignores the word limit): Truncate at 300 words after
  generation.
- **Summarization triggered every turn** (missing `summary_updated_at` check):
  Set `state["summary_updated_at"] = time.time()` after summarization. Check
  this key to prevent re-triggering.
- **Previous summary is wrong** (LLM hallucinated a fact): Store a `summary_version`
  in state. Run summarization with a different prompt or higher temperature if
  quality is consistently poor.

## Examples

### Example 1 — Auto-triggered summarization in before_agent_callback

```python
import time
import google.generativeai as genai
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content

SUMMARY_TURN_THRESHOLD = 20
SUMMARY_COOLDOWN_SECONDS = 300  # Don't re-summarize within 5 minutes

def auto_summarize_callback(callback_context: CallbackContext) -> Optional[Content]:
    """Auto-triggers history summarization when the turn count exceeds the threshold."""
    events = callback_context._invocation_context.session.events
    last_summary_time = float(callback_context.state.get("summary_updated_at", 0))

    if (len(events) > SUMMARY_TURN_THRESHOLD and
            time.time() - last_summary_time > SUMMARY_COOLDOWN_SECONDS):
        # Collect history from oldest events
        lines = []
        for event in events[:-5]:  # Summarize all but the last 5 events
            if event.content and event.content.parts:
                role = "User" if event.author == "user" else "Agent"
                text = " ".join(p.text for p in event.content.parts if hasattr(p, "text") and p.text)
                if text:
                    lines.append(f"{role}: {text[:300]}")

        if lines:
            history = "\n".join(lines[:20])
            prompt = (
                "Summarize this conversation in ≤150 words. "
                "Preserve: user goals, decisions made, bookings confirmed, preferences stated, next steps.\n\n"
                + history
            )
            try:
                response = genai.generate_content(
                    model="gemini-2.0-flash-lite",
                    contents=[{"parts": [{"text": prompt}]}]
                )
                summary = (response.text or "").strip()[:1500]  # Hard cap at 1500 chars
                callback_context.state["session_summary"] = summary
                callback_context.state["user:last_session_summary"] = summary
                callback_context.state["summary_updated_at"] = time.time()
            except Exception as exc:
                import logging
                logging.getLogger("summarization").error(f"Summarization failed: {exc}")

    return None  # Don't block agent invocation
```

### Example 2 — Structured summary with Pydantic schema

```python
from pydantic import BaseModel, Field
from typing import List
from google.adk.agents import LlmAgent

class SessionSummary(BaseModel):
    user_goal: str = Field(description="Primary goal the user is trying to accomplish")
    completed_actions: List[str] = Field(description="Actions completed so far")
    pending_actions: List[str] = Field(description="Actions still pending")
    key_facts: List[str] = Field(description="Important facts: bookings, prices, preferences")
    next_recommended_step: str = Field(description="Recommended next step for the agent")

SUMMARIZER = LlmAgent(
    name="session_summarizer",
    model="gemini-2.0-flash-lite",
    instruction="Summarize the session history stored in state. Populate all schema fields accurately.",
    output_schema=SessionSummary,
    output_key="structured_session_summary",
)
```

## Edge Cases

- **Recursive summarization** (summary of summary): Only summarize the raw
  event history, not a previous summary. Track what has been summarized with
  `summary_covers_events_0_to_N` in state.
- **Session begins with a summary** (continuation from previous session):
  The summary should be the first context chunk in the instruction, before
  current conversation events.
- **Summarization changes factual details** (LLM hallucination): Test summaries
  against known fact sets. Add explicit: "Do not add facts not present in the
  source text" to the summarization prompt.

## Integration Notes

- **`google-adk-context-window-management`**: Summarization is the primary tool
  for freeing context window space in long-running sessions.
- **`google-adk-memory-compression`**: Summarization is the LLM-based variant
  of compression. Use structural compaction for structured data.
- **`google-adk-episodic-memory`**: Store post-session summaries as an episode
  in the user's episodic memory log for later retrieval.
- **`google-adk-long-term-memory`**: Cross-session summaries are stored in
  `user:` state and retrieved at the start of new sessions.
