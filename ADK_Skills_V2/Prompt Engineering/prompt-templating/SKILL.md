---
name: prompt-templating
description: >
  Use this skill when constructing reusable, parameterized prompt templates for
  Google ADK LlmAgents. Covers using ADK's {key} and {key?} state injection
  syntax in instruction strings, building prompt template libraries, combining
  static and dynamic template sections, and managing template versioning
  for production ADK deployments.
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

This skill implements reusable prompt templates for Google ADK agents. Instead
of hardcoding unique prompts per agent instance, prompt templating creates a
modular library of reusable prompt sections (role, rules, format, examples)
that are parameterized via ADK's `{key}` state injection syntax and assembled
at runtime. This enables prompt reuse across multiple agents, environment-specific
prompt variants, and safe dynamic content injection without hardcoding.

## When to Use

- When multiple `LlmAgent` instances share common prompt sections (role
  definition, safety rules, output format) that should be maintained in one place.
- When the agent prompt changes based on deployment environment, user segment,
  or feature flags stored in `app:` or `meta:` scoped state.
- When building a prompt library for a multi-agent system with dozens of agents.
- When prompt changes should be managed centrally without modifying agent code.
- When dynamic content (user name, session context, step) must be injected into
  a static prompt structure.

## When NOT to Use

- Do not use template injection for secret data (API keys, tokens) — use secure
  environment variables instead.
- Do not inject large objects (lists, dicts) directly via `{key}` — serialize
  to a concise string first.
- Do not use templates for fully dynamic prompts where the structure itself
  varies completely — use `InstructionProvider` instead.

## Google ADK Context

- **`{key}` syntax**: ADK resolves `{key}` placeholders in the `instruction`
  string by looking up `session.state["key"]`. If the key is missing, an error
  is raised.
- **`{key?}` syntax**: Optional injection. If `session.state["key"]` is absent
  or None, the placeholder is replaced with an empty string.
- **`user:key?`**: Injects user-scoped state (e.g., `{user:name?}`, `{user:role?}`).
- **`app:key?`**: Injects app-scoped state (e.g., `{app:version?}`, `{app:feature_flag?}`).
- **`InstructionProvider`**: For templates that require conditional logic or
  complex assembly, use a Python function `(ReadonlyContext) -> str`.
- **ADK Agent Execution Lifecycle**: Template resolution happens once per
  invocation, before the LLM call. State values at the time of invocation are
  used.
- **Python f-strings + `LlmAgent.instruction`**: For templates assembled in
  Python code (not relying on ADK's `{key}` injection), use Python f-strings
  to compose the instruction at agent creation time.

## Capabilities

- Defines reusable prompt section templates (role, rules, format, examples).
- Injects session state into instruction strings via `{key?}` placeholders.
- Assembles multi-section instructions from a template library.
- Manages environment-specific prompt variants (dev/staging/prod).
- Implements template versioning for production prompt management.
- Uses `InstructionProvider` for conditional template assembly.
- Validates template variables at agent initialization.

## Step-by-Step Instructions

1. **Define reusable template sections** as string constants:
   ```python
   ROLE_TEMPLATE = "You are {agent_role?} for {company_name?}."
   RULES_TEMPLATE = """
   RULES:
   - Always call {primary_tool?} before any action.
   - Respond in {user:preferred_language?}.
   - Maximum response length: {max_response_words?} words.
   """
   FORMAT_TEMPLATE = "Present results as a numbered list with: [N]. {fields_format?}"
   ```

2. **Assemble the full instruction** using Python string concatenation:
   ```python
   FULL_INSTRUCTION = f"{ROLE_TEMPLATE}\n{RULES_TEMPLATE}\n{FORMAT_TEMPLATE}"
   ```

3. **Initialize session state** with template variable values:
   ```python
   session = await session_service.create_session(
       app_name="my_app",
       user_id=user_id,
       state={
           "agent_role": "flight booking specialist",
           "company_name": "AcmeAir",
           "primary_tool": "search_flights",
           "max_response_words": "150",
           "fields_format": "[Airline] [Flight#] | [Time] | $[Price]",
           "user:preferred_language": "English",
       }
   )
   ```

4. **Register the template instruction on the agent**:
   ```python
   agent = LlmAgent(
       name="templated_agent",
       model="gemini-2.0-flash",
       instruction=FULL_INSTRUCTION,
       tools=[search_flights],
   )
   ```

5. **For conditional templates**, use `InstructionProvider`:
   ```python
   def template_provider(ctx: ReadonlyContext) -> str:
       env = ctx.state.get("meta:env", "prod")
       base = PROD_TEMPLATE if env == "prod" else DEV_TEMPLATE
       return base.format(
           user_name=ctx.state.get("user:name", "Valued Customer"),
           step=ctx.state.get("booking_step", "search"),
       )
   ```

6. **Version-control prompt templates** in a `prompts/` directory with semantic
   versioning. Store the active version in `app:prompt_version` state.

7. **Validate templates** at startup by checking all `{key}` placeholders have
   corresponding state entries.

## Input Format

```python
{
  "agent_goal": "Reusable templated flight booking agent",
  "user_input": "Find flights to Tokyo",
  "context": {
    "agent_role": "flight booking specialist",
    "company_name": "AcmeAir",
    "primary_tool": "search_flights",
    "user:name": "Alice",
    "user:preferred_language": "English"
  },
  "tools": ["search_flights", "process_payment"],
  "output_format": "numbered list with airline, time, price",
  "constraints": ["Must call primary_tool first", "Max 150 words"]
}
```

## Output Format

```python
{
  "system_prompt": "You are flight booking specialist for AcmeAir.\nRULES: Always call search_flights...",
  "instructions": "...",
  "expected_output_format": {"type": "numbered_list"},
  "tool_usage_guidance": {"search_flights": "Must call first"}
}
```

## Error Handling

- **Missing required `{key}` value** (KeyError from ADK): Use `{key?}` for
  optional injections. For required values, validate state at session creation.
- **Wrong type injected** (dict instead of str): ADK calls `str()` on all
  injected values. Ensure state values intended for injection are simple strings.
- **Template variable name collision** (same key used in multiple sections
  with different meanings): Use namespaced key names: `{rules:primary_tool?}`
  vs. `{format:primary_tool?}`.
- **Template becomes stale** (instructions reference a removed tool): Validate
  templates against the current tool list at agent startup.
- **Template too long after injection**: Measure token count after injection.
  Fail fast at startup if the injected instruction exceeds the token limit.

## Examples

### Example 1 — Reusable template library

```python
from google.adk.agents import LlmAgent

# --- Template Sections ---

ROLE_TEMPLATE = """
You are {agent_role?} for {company_name?}.
You are assisting {user:name?} (Language: {user:preferred_language?}).
"""

TOOL_RULES_TEMPLATE = """
TOOL PROTOCOL:
1. Always call {step_1_tool?} FIRST to gather information.
2. Present results clearly. Ask the user to confirm before any action.
3. Call {step_2_tool?} ONLY after explicit user confirmation.
"""

FORMAT_TEMPLATE = """
OUTPUT FORMAT:
- Results: Numbered list. Template: "[N]. {item_format?}"
- Confirmations: Single paragraph, max 60 words.
- Errors: Start with "I'm sorry, " + issue + next step.
- Max response length: {max_words?} words.
"""

SAFETY_TEMPLATE = """
SAFETY:
- Stay strictly within scope: {agent_scope?}.
- Decline all out-of-scope requests politely.
- Never invent data — only use tool results.
"""

# --- Full Template Assembly ---
FLIGHT_AGENT_TEMPLATE = f"{ROLE_TEMPLATE}\n{TOOL_RULES_TEMPLATE}\n{FORMAT_TEMPLATE}\n{SAFETY_TEMPLATE}"

# --- Agent using the template ---
flight_agent = LlmAgent(
    name="flight_booking_agent",
    model="gemini-2.0-flash",
    instruction=FLIGHT_AGENT_TEMPLATE,
    tools=[search_flights, process_payment],
)

# --- Session state initializes template variables ---
async def create_flight_session(session_service, user_id: str):
    return await session_service.create_session(
        app_name="flight_app",
        user_id=user_id,
        state={
            "agent_role": "flight booking specialist",
            "company_name": "AcmeAir",
            "step_1_tool": "search_flights",
            "step_2_tool": "process_payment",
            "item_format": "[Airline] [Flight#] | [Departs→Arrives] | [Duration] | $[Price]",
            "max_words": "150",
            "agent_scope": "flight search and booking",
            "user:name": "Alice",
            "user:preferred_language": "English",
        },
    )
```

### Example 2 — Conditional template via InstructionProvider

```python
from google.adk.agents.readonly_context import ReadonlyContext

TEMPLATES_BY_ENV = {
    "prod": """
You are FlightBot for AcmeAir. {user:name?}, I'm ready to help you book flights.
RULES: Call search_flights first. Confirm before payment. Max 150 words.
""",
    "dev": """
[DEV MODE] You are FlightBot. Debug logging is active.
Session: {meta:session_id?}. User: {user:name?}.
RULES: Same as prod, but also log tool reasoning.
""",
}

def env_template_provider(ctx: ReadonlyContext) -> str:
    env = ctx.state.get("meta:env", "prod")
    return TEMPLATES_BY_ENV.get(env, TEMPLATES_BY_ENV["prod"])
```

### Example 3 — Template validation at startup

```python
import re

def validate_template(template: str, required_state_keys: list[str]) -> list[str]:
    """Validates that all required {key} placeholders have a corresponding state key."""
    injected_keys = re.findall(r'\{(\w[\w:]*)\??}', template)
    required_injected = [k for k in injected_keys if not k.endswith("?")]
    missing = [k for k in required_injected if k not in required_state_keys]
    return missing

missing_keys = validate_template(FLIGHT_AGENT_TEMPLATE, list(session_state.keys()))
if missing_keys:
    raise ValueError(f"Template references missing state keys: {missing_keys}")
```

## Edge Cases

- **Template injection with special regex chars** (e.g., `{user:name?}` = "Alice & Bob"):
  ADK escapes injection values. No action needed for standard strings.
- **Template shared across agents with different tools**: Each agent has its
  own tool list. The `{step_1_tool?}` must match actual tools registered on that
  agent. Validate at startup.
- **Template versioning conflict**: Store version in `app:prompt_version` state.
  Log the version with every invocation for auditability.
- **Template too large for flash model**: Keep templates under 2000 tokens.
  Extract rarely-needed sections to on-demand injection via `InstructionProvider`.

## Integration Notes

- **`google-adk-context-injection`**: The `{key?}` injection mechanism is the
  ADK context injection system. This skill builds template structures on top of it.
- **`google-adk-dynamic-prompt-generation`**: For templates that require
  programmatic assembly logic beyond simple `{key?}` injection, use that skill.
- **`google-adk-system-prompts`**: Templates assemble the full system prompt.
  The template library replaces hardcoded strings in the `system-prompts` skill.
- **`app:` scoped state**: Store global template variables (company name, version,
  feature flags) in `app:` state so they are accessible across all sessions.
