---
name: dynamic-prompt-generation
description: >
  Use this skill when a Google ADK LlmAgent requires a prompt that changes
  based on runtime state, user context, session history, or workflow step.
  Covers implementing InstructionProvider functions, selecting prompt variants
  from session.state, assembling prompts programmatically, and injecting
  real-time context into the agent instruction on every invocation.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: prompt-engineering
---

## Purpose

This skill implements dynamic prompt generation for Google ADK `LlmAgent` —
constructing the agent's instruction at runtime based on the current session
state, user profile, workflow step, or external context. While a static
instruction handles most cases, dynamic prompt generation is required when the
agent must adapt its core instruction, tool usage guidance, or behavioral
constraints to the specific context of each invocation. ADK's `InstructionProvider`
is the primary mechanism for this capability.

## When to Use

- When the agent's behavior must change significantly based on session state
  (e.g., different step in a multi-step workflow).
- When different user segments require fundamentally different agent behaviors
  (e.g., admin vs. regular user).
- When external context (A/B test variant, feature flag, real-time configuration)
  must be injected into the instruction at runtime.
- When the conversation summary or history must be included in the instruction
  and grows over time.
- When `{key?}` static injection is insufficient because the prompt structure
  itself (not just values) must change.

## When NOT to Use

- Do not use `InstructionProvider` when static `{key?}` state injection is
  sufficient — static injection is simpler and more maintainable.
- Do not make external API calls inside `InstructionProvider` — it runs
  synchronously during the invocation setup phase. Pre-fetch and cache
  external data in `session.state`.
- Do not generate prompts with unbounded length — always cap dynamic sections
  at a token budget.

## Google ADK Context

- **`InstructionProvider`**: A Python callable `(ReadonlyContext) -> str` passed
  as `LlmAgent.instruction`. Called once per invocation. Returns the fully
  assembled instruction string for that invocation.
- **`ReadonlyContext`**: Provides read-only access to `session.state`, `user_id`,
  `session_id`, and `app_name` within the provider.
- **`session.state`**: The source of all dynamic context in the provider.
  Pre-load relevant data (user profile, step, flags) into state before invocation.
- **`{key?}` injection**: For simple value injection, use ADK's built-in `{key?}`
  syntax. Use `InstructionProvider` only when conditional logic or programmatic
  assembly is required.
- **`before_agent_callback`**: Use to pre-compute expensive context (e.g.,
  conversation summaries) and store in state before `InstructionProvider` runs.
- **ADK Agent Execution Lifecycle**:
  1. Session is loaded.
  2. `before_agent_callback` runs (optional).
  3. `InstructionProvider` is called → returns instruction string.
  4. LLM call is made with the instruction.
- **Token budget**: Dynamic prompts can grow unbounded. Measure and cap at
  2000 tokens for flash models, 4000 for pro models.

## Capabilities

- Implements `InstructionProvider` functions for runtime prompt assembly.
- Selects instruction variant based on workflow step, user role, or feature flags.
- Injects conversation summaries, user profiles, and real-time context dynamically.
- Assembles multi-section prompts conditionally based on state.
- Manages prompt token budget dynamically using character limits.
- Reads `app:`, `user:`, and session-scoped state in the provider.

## Step-by-Step Instructions

1. **Identify what must change** in the instruction per invocation:
   - Workflow step → different task guidance
   - User role → different permissions/capabilities
   - Feature flag → different tool availability
   - Conversation summary → evolving context injection

2. **Pre-load dynamic context** into `session.state` (via session creation or
   `before_agent_callback`):
   ```python
   session = await session_service.create_session(
       app_name="my_app", user_id=user_id,
       state={"booking_step": "search", "user:role": "premium", "app:feature_v2": True}
   )
   ```

3. **Implement the `InstructionProvider`**:
   ```python
   from google.adk.agents.readonly_context import ReadonlyContext

   def my_instruction_provider(ctx: ReadonlyContext) -> str:
       step = ctx.state.get("booking_step", "search")
       role = ctx.state.get("user:role", "standard")
       # ... conditional assembly logic ...
       return assembled_instruction
   ```

4. **Register the provider on the agent**:
   ```python
   agent = LlmAgent(
       name="dynamic_agent",
       model="gemini-2.0-flash",
       instruction=my_instruction_provider,
       tools=[...],
   )
   ```

5. **Measure instruction length** in the provider:
   ```python
   if len(assembled_instruction.split()) > 400:  # ~2000 tokens estimate
       assembled_instruction = assembled_instruction[:3000]  # Char limit
   ```

6. **Test each state branch** that produces a different instruction variant
   by running eval cases for each scenario.

7. **Cache expensive computations** in state before the provider runs rather
   than computing them inside the provider.

## Input Format

```python
{
  "agent_goal": "Step-adaptive booking agent instruction",
  "user_input": "I want to book the first option",
  "context": {
    "booking_step": "confirm",
    "selected_flight": "BA 178",
    "user:name": "Alice",
    "user:role": "premium",
    "app:feature_v2": True
  },
  "tools": ["search_flights", "process_payment"],
  "output_format": "step-appropriate response",
  "constraints": ["Instruction must reflect current workflow step"]
}
```

## Output Format

```python
{
  "system_prompt": "[dynamically assembled for step=confirm, role=premium]...",
  "instructions": "...",
  "expected_output_format": {"step": "confirm", "action": "present_and_confirm"},
  "tool_usage_guidance": {
    "process_payment": "Available only after user confirms with 'yes'"
  }
}
```

## Error Handling

- **`InstructionProvider` raises exception**: Catch all exceptions inside the
  provider and return a safe fallback instruction. Never let the provider crash.
  ```python
  def safe_provider(ctx: ReadonlyContext) -> str:
      try:
          return build_instruction(ctx)
      except Exception as e:
          import logging
          logging.getLogger("prompt").error(f"InstructionProvider error: {e}")
          return FALLBACK_INSTRUCTION
  ```
- **Missing state key in provider**: Use `.get(key, default)` — never direct
  dict access. Missing keys are common and must be handled gracefully.
- **Dynamic instruction too long**: Add a hard character limit in the provider
  and a warning log when truncation occurs.
- **Provider returns empty string**: Raise promptly at provider level —
  `assert assembled_instruction, "Provider returned empty instruction"`.
- **State changes mid-invocation** (tool updates state): The instruction is
  resolved once before the LLM call. State changes during the turn don't update
  the instruction for that turn.

## Examples

### Example 1 — Step-adaptive workflow instruction provider

```python
from google.adk.agents import LlmAgent
from google.adk.agents.readonly_context import ReadonlyContext
import logging

logger = logging.getLogger("prompt")

STEP_INSTRUCTIONS = {
    "search": """
Your current task: Help {user:name?} search for available flights.
REQUIRED: Call search_flights(origin, destination, date) before showing any options.
After receiving results, present them as a numbered list with airline, time, duration, and price.
""",
    "confirm": """
Your current task: Present the selected flight to {user:name?} and ask for confirmation.
Selected flight: {selected_flight?}
Present all booking details clearly. Ask: "Shall I proceed with booking? Reply 'yes' to confirm."
Do NOT call process_payment until the user replies 'yes'.
""",
    "payment": """
Your current task: Process the booking for {user:name?}.
The user has confirmed their selection. Call process_payment now.
After payment, provide: booking reference, flight details, and payment amount.
""",
    "complete": """
Your current task: Confirm the completed booking to {user:name?} and offer help.
Booking reference: {booking_reference?}
Ask if there is anything else you can help with.
""",
}

BASE_INSTRUCTION = """
You are FlightBot for AcmeAir.
{user:name?}, I'm your dedicated flight booking assistant.
Respond in {user:preferred_language?}. Stay strictly within flight booking scope.
"""

def booking_instruction_provider(ctx: ReadonlyContext) -> str:
    """Dynamically assembles instruction based on the current booking workflow step."""
    try:
        step = ctx.state.get("booking_step", "search")
        step_instruction = STEP_INSTRUCTIONS.get(step, STEP_INSTRUCTIONS["search"])

        # Inject feature flag for V2 search
        tool_note = ""
        if ctx.state.get("app:feature_search_v2", False):
            tool_note = "\nNote: Use search_flights_v2 for enhanced results."

        assembled = f"{BASE_INSTRUCTION}\n{step_instruction}{tool_note}"

        # Token budget guard
        if len(assembled) > 3000:
            logger.warning(f"Instruction exceeds 3000 chars. Truncating. Step: {step}")
            assembled = assembled[:3000]

        return assembled

    except Exception as exc:
        logger.error(f"InstructionProvider failed: {exc}. Using fallback.")
        return "You are FlightBot for AcmeAir. Help the user with flight booking."


flight_agent = LlmAgent(
    name="adaptive_flight_agent",
    model="gemini-2.0-flash",
    instruction=booking_instruction_provider,
    tools=[search_flights, process_payment],
)
```

### Example 2 — Role-adaptive instruction provider

```python
def role_adaptive_provider(ctx: ReadonlyContext) -> str:
    """Adjusts agent capabilities based on the user's role."""
    role = ctx.state.get("user:role", "standard")
    name = ctx.state.get("user:name", "Valued Customer")

    if role == "admin":
        capabilities = """
ADMIN CAPABILITIES:
- Access all customer bookings via get_all_bookings.
- Modify or cancel any booking via admin_modify_booking.
- View system reports via get_admin_report.
"""
    elif role == "premium":
        capabilities = """
PREMIUM CAPABILITIES:
- Priority search results via search_flights_priority.
- Seat upgrade options via request_seat_upgrade.
"""
    else:  # standard
        capabilities = """
STANDARD CAPABILITIES:
- Search flights via search_flights.
- Book flights via process_payment.
"""

    return f"""
You are FlightBot assisting {name} (role: {role}).
{capabilities}
Stay within scope. Respond professionally and concisely (max 150 words).
"""
```

### Example 3 — Conversation summary injection via before_agent_callback

```python
from google.adk.agents.callback_context import CallbackContext
from google.adk.agents import LlmAgent

def inject_summary_before_agent(callback_context: CallbackContext):
    """Pre-computes and stores conversation summary before InstructionProvider runs."""
    events = callback_context._invocation_context.session.events
    if len(events) > 20 and "conversation_summary" not in callback_context.state:
        # Trigger summarization (simplified example)
        recent_texts = [
            p.text for e in events[-10:]
            if e.content and e.content.parts
            for p in e.content.parts
            if hasattr(p, "text") and p.text
        ]
        summary = " ".join(recent_texts)[:500]
        callback_context.state["conversation_summary"] = summary
    return None

def summary_aware_provider(ctx: ReadonlyContext) -> str:
    summary = ctx.state.get("conversation_summary", "")
    summary_section = f"\nCONTEXT SUMMARY:\n{summary[:300]}\n" if summary else ""
    return f"""
You are FlightBot for AcmeAir.{summary_section}
Help the user with their flight booking request.
Always call search_flights before quoting prices. Max 150 words.
"""

agent = LlmAgent(
    name="summary_aware_agent",
    model="gemini-2.0-flash",
    instruction=summary_aware_provider,
    before_agent_callback=inject_summary_before_agent,
    tools=[search_flights, process_payment],
)
```

## Edge Cases

- **Infinite loop of state-triggered instruction changes**: `InstructionProvider`
  runs once per invocation. It cannot loop. State changes during a turn do not
  re-invoke the provider.
- **Heavy computation in provider** (database query inside provider): Pre-compute
  in `before_agent_callback` and cache in state. Never do I/O inside the provider.
- **Provider returns different instructions for the same state on retries**:
  The provider is deterministic for a given state snapshot. Non-determinism
  indicates a bug in the provider logic.
- **Feature flag changes mid-session** (deployed while session is active):
  The next invocation will pick up the new flag value from `app:` state. No
  session restart needed.

## Integration Notes

- **`google-adk-prompt-templating`**: Dynamic prompt generation uses template
  sections assembled conditionally. Combine with `prompt-templating` for
  modular provider design.
- **`google-adk-context-injection`**: Static `{key?}` injection is the simpler
  alternative. Use `InstructionProvider` only when static injection is insufficient.
- **`before_agent_callback`**: The pre-computation hook for dynamic prompts.
  Expensive operations belong here, not in the provider.
- **`google-adk-context-summarization`**: The summary stored in state is the
  primary dynamic content injected via providers for long conversations.
