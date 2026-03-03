---
name: google-adk-context-injection
description: >
  Use this skill when a Google ADK LlmAgent needs to inject session state
  values, user data, or dynamic content directly into its instruction string
  at runtime. Covers {key} placeholder templating, InstructionProvider
  functions, inject_session_state utility, and ReadonlyContext access patterns.
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

This skill enables `LlmAgent` to dynamically populate its `instruction` string
with live session state values before each model call. Instead of relying on
the LLM to infer user-specific context from conversation history, this skill
ensures relevant state (user name, current step, preferences) is explicitly
injected into the system prompt — making agent behavior deterministic and
context-aware.

## When to Use

- When an `LlmAgent` instruction must reference a session state value that
  changes across turns (e.g., `{booking_step}`, `{user:name}`).
- When the instruction contains optional state values that may not always exist.
- When the instruction contains literal curly braces (JSON examples) and you
  must use an `InstructionProvider` to avoid unintended injection.
- When building dynamic instructions that depend on complex state logic.

## When NOT to Use

- Do not use `{key}` templating when the key may not exist and you haven't
  suffixed it with `?` (e.g., use `{topic?}` not `{topic}` for optional keys).
- Do not use string instruction templating when the instruction contains JSON
  or format strings with literal `{}` — use `InstructionProvider` instead.
- Do not use this skill to inject secrets or credentials into instructions —
  use environment variables accessed in tool functions instead.

## Google ADK Context

- **`session.state` → `{key}` injection**: ADK automatically replaces `{key}`
  placeholders in `LlmAgent.instruction` strings with values from `session.state`
  before passing the instruction to the LLM.
- **Optional keys**: `{key?}` — if the key is missing from state, the
  placeholder is replaced with an empty string rather than raising an error.
- **`InstructionProvider`**: A callable `(ReadonlyContext) -> str` passed as the
  `instruction` parameter. ADK does NOT perform `{key}` injection on provider-
  returned strings by default — you must explicitly call `inject_session_state`.
- **`ReadonlyContext`**: Provides read-only access to `session.state`, `session_id`,
  `user_id`, and `app_name` within an `InstructionProvider`.
- **`inject_session_state` utility**: From `google.adk.utils.instructions_utils`.
  Manually applies `{key}` replacement on a template string within a provider.
- **ADK Execution Lifecycle**: Instruction injection occurs once per agent
  invocation, before the LLM call.

## Capabilities

- Inject session state values into `LlmAgent` instructions via `{key}` syntax.
- Use `{key?}` for optional state fields that may not always be present.
- Build fully dynamic instructions via `InstructionProvider` callables.
- Combine provider-generated instructions with explicit state injection via
  `inject_session_state`.
- Access `tool_context.user_id` and `tool_context.session_id` in tools to
  construct context-aware inputs.

## Step-by-Step Instructions

1. **Identify which state values the instruction needs**: List all `session.state`
   keys the instruction references.

2. **Ensure keys exist in session state** before the agent run (initialize in
   `create_session(state={...})` or via previous tool calls).

3. **Use `{key}` for required state fields** in `LlmAgent.instruction`:
   ```python
   LlmAgent(instruction="User: {user:name}. Step: {booking_step}.")
   ```

4. **Use `{key?}` for optional state fields** to avoid errors on missing keys:
   ```python
   LlmAgent(instruction="Discount code: {user:discount_code?}.")
   ```

5. **Use `InstructionProvider`** when the instruction contains literal `{}`:
   ```python
   def my_provider(ctx: ReadonlyContext) -> str:
       step = ctx.state.get("booking_step", "start")
       return f'Guide the user through step: {step}. Output JSON: {{"status": ""}}'
   LlmAgent(instruction=my_provider)
   ```

6. **Combine provider with state injection** for hybrid use:
   ```python
   from google.adk.utils import instructions_utils
   async def dynamic_provider(ctx: ReadonlyContext) -> str:
       template = "Help with {booking_step}. Format: {\"key\": \"val\"}"
       return await instructions_utils.inject_session_state(template, ctx)
   ```

7. **Use `ReadonlyContext`** in providers to access additional context:
   ```python
   def provider(ctx: ReadonlyContext) -> str:
       user_id = ctx.user_id
       name = ctx.state.get("user:name", "User")
       return f"You are assisting {name} (ID: {user_id})."
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_xyz",
  "state": {
    "booking_step": "select_seat",
    "user:name": "Alice",
    "user:discount_code": "SAVE10"
  },
  "metadata": {}
}
```

## Output Format

```python
{
  "state_updated": False,
  "injected_instruction": "User: Alice. Step: select_seat. Discount: SAVE10.",
  "missing_optional_keys": []
}
```

## Error Handling

- **`{key}` key missing from state and not optional**: ADK raises a `KeyError`.
  Fix by either initializing the key in `create_session(state={...})` or changing
  to `{key?}`.
- **Literal `{}` in instruction string treated as placeholder**: Switch to an
  `InstructionProvider` function to avoid unintended substitution.
- **`InstructionProvider` raises exception**: Exception propagates to the runner.
  Wrap provider body in try/except and return a safe fallback instruction.
- **State value is not a string**: ADK converts to string via `str()`. Document
  expected types to avoid confusing LLM inputs.
- **`inject_session_state` with invalid key names**: Keys must be valid Python
  identifiers. Numeric-only keys or keys with hyphens will not be injected.

## Examples

### Example 1 — Basic state injection

```python
from google.adk.agents import LlmAgent
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai.types import Content, Part

booking_agent = LlmAgent(
    name="booking_agent",
    model="gemini-2.0-flash",
    instruction=(
        "You are helping {user:name} book a flight. "
        "Current step: {booking_step}. "
        "Preferred language: {user:preferred_language?}."
    ),
)

session_service = InMemorySessionService()
session = await session_service.create_session(
    app_name="booking_app",
    user_id="user_001",
    state={
        "user:name": "Alice",
        "booking_step": "select_flight",
        "user:preferred_language": "English",
    },
)
runner = Runner(agent=booking_agent, app_name="booking_app",
                session_service=session_service)
```

### Example 2 — InstructionProvider with literal braces

```python
from google.adk.agents import LlmAgent
from google.adk.agents.readonly_context import ReadonlyContext

def booking_instruction_provider(ctx: ReadonlyContext) -> str:
    step = ctx.state.get("booking_step", "start")
    name = ctx.state.get("user:name", "User")
    # Literal {  } in JSON example won't be replaced — provider disables injection
    return (
        f"You are helping {name} with step: {step}. "
        'Respond with JSON: {"step": "", "message": ""}'
    )

agent = LlmAgent(
    name="booking_agent",
    model="gemini-2.0-flash",
    instruction=booking_instruction_provider,
)
```

### Example 3 — Provider with explicit inject_session_state

```python
from google.adk.agents.readonly_context import ReadonlyContext
from google.adk.utils import instructions_utils

async def hybrid_provider(ctx: ReadonlyContext) -> str:
    template = (
        "Help {user:name} complete {booking_step}. "
        'Output: {"confirmed": true}'  # Literal braces — safe because no {key} pattern
    )
    return await instructions_utils.inject_session_state(template, ctx)
```

## Edge Cases

- **f-string instruction with `{{key}}`**: Python f-strings convert `{{key}}`
  to `{key}` at parse time. ADK then injects the state value. This is valid but
  only works with f-strings — avoid mixing f-string and provider patterns.
- **State value changes mid-conversation**: Injection happens once per invocation.
  The LLM sees the state value as of the start of the current turn.
- **Very large state values injected into instruction**: Long injected values
  inflate the system prompt. Keep injected values concise (< 200 chars).
- **Provider returns empty string**: ADK passes the empty string to the LLM,
  resulting in an unguided response. Always return a non-empty fallback.

## Integration Notes

- **`google-adk-conversation-state-tracking`**: Provides the `session.state`
  values that this skill injects. Apply state-tracking skill first.
- **`google-adk-user-specific-context`**: Use `user:` prefixed keys in injection
  for user-specific personalization across sessions.
- **`google-adk-context-pruning`**: If injected context grows too large, apply
  pruning before injection to stay within token limits.
- **ADK `output_key`**: Complements injection — `output_key` writes agent
  responses back to state, which can then be injected in subsequent turns.
