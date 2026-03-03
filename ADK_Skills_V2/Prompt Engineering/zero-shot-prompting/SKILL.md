---
name: zero-shot-prompting
description: >
  Use this skill when a Google ADK LlmAgent must perform a task relying solely
  on clear, precise instructions without any worked examples. Covers writing
  high-quality zero-shot instructions that are specific enough to guide correct
  agent behavior, tool selection, and output formatting without demonstration
  examples in the instruction field.
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

This skill designs zero-shot prompts for Google ADK `LlmAgent` — instructions
that guide correct behavior without relying on embedded worked examples. Zero-shot
prompting is the default starting point for new ADK agents. It relies on the
precision and clarity of the instruction language to produce correct tool call
sequences, output formats, and reasoning behavior. When examples are not feasible
(due to token budget, task diversity, or privacy constraints), zero-shot
instructions must be crafted with maximum clarity, specificity, and structure.

## When to Use

- When starting a new `LlmAgent` and writing the initial instruction before
  deciding whether few-shot examples are needed.
- When token budget is tight and examples would exceed the limit.
- When the task is diverse enough that pre-written examples would not generalize.
- When the LLM's base capabilities are sufficient and examples would not add value.
- When few-shot prompting is impossible due to PII in examples.

## When NOT to Use

- Do not rely on zero-shot when the agent consistently fails on a specific
  output pattern — add few-shot examples for that pattern.
- Do not use zero-shot for highly specialized tasks with non-standard output
  requirements — the model's priors may not align.
- Do not equate zero-shot with minimal instruction — zero-shot still requires
  a detailed, precise instruction; it just lacks worked examples.

## Google ADK Context

- **`LlmAgent.instruction`**: Zero-shot prompts are pure instruction strings
  without embedded `USER:` / `AGENT:` example pairs.
- **Instruction precision**: In zero-shot, every word matters. Vague instructions
  ("help users") produce inconsistent results. Precise instructions ("call
  search_flights before quoting prices") produce reliable results.
- **ADK Tool Docstrings**: Critical for zero-shot success. When no example
  demonstrates how a tool should be called, the model relies entirely on the
  tool's docstring to understand when and how to invoke it.
- **`output_schema`**: A Pydantic output schema is the most reliable way to
  enforce output format in zero-shot mode — more reliable than textual
  formatting instructions alone.
- **`{key?}` injection**: Zero-shot instructions benefit from dynamic context
  injection. The instruction is precise but adapts to the current session state.
- **ADK Agent Execution Lifecycle**: The zero-shot instruction is passed as
  the system prompt. No examples are in the context — the model must generalize
  from instructions alone.

## Capabilities

- Writes precise, example-free instructions for ADK agents.
- Structures zero-shot instructions with clear role, rules, tool guidance,
  and output format sections.
- Enforces output format via `output_schema` instead of textual examples.
- Maximizes instruction clarity through imperative, structured language.
- Combines zero-shot instructions with dynamic state injection for precision.
- Identifies when zero-shot is insufficient and few-shot is needed.

## Step-by-Step Instructions

1. **Write a precise role statement** (not vague):
   ```
   ✗ VAGUE: "You are a helpful assistant."
   ✓ PRECISE: "You are a customer order management agent for RetailBot."
   ```

2. **Define task scope with explicit action verbs**:
   ```
   Your task is to: (1) verify order status using check_order, (2) process
   refunds using process_refund, and (3) escalate unresolvable issues using
   escalate_ticket.
   ```

3. **Write tool usage guidance without examples**:
   ```
   TOOL PROTOCOL:
   - check_order(order_id): Call FIRST for any order inquiry. Required before
     all other tools.
   - process_refund(order_id, reason): Call ONLY if check_order confirms the
     order is eligible for refund (status: "delivered" or "damaged").
   - escalate_ticket(order_id, issue_summary): Call ONLY if process_refund
     fails or if the issue requires human intervention.
   ```

4. **Define output format with concrete structure**:
   ```
   OUTPUT FORMAT:
   Begin with a one-sentence status summary.
   Follow with the action taken.
   End with a clear next step for the user.
   Maximum 100 words.
   ```

5. **Add a zero-shot constraint checklist**:
   ```
   VERIFY BEFORE RESPONDING:
   [ ] Did you call check_order before any action?
   [ ] Is your response under 100 words?
   [ ] Does your response include a next step for the user?
   ```

6. **Use `output_schema`** for structured output enforcement:
   ```python
   class OrderResponse(BaseModel):
       status_summary: str
       action_taken: str
       next_step: str
   ```

7. **Test extensively** — zero-shot requires more testing than few-shot since
   the model has no anchoring examples. Run 20+ test cases.

## Input Format

```python
{
  "agent_goal": "Handle customer order inquiries and refunds",
  "user_input": "My order hasn't arrived yet",
  "context": {"order_id": "ORD-789", "user:name": "Bob"},
  "tools": ["check_order", "process_refund", "escalate_ticket"],
  "output_format": "status_summary + action + next_step (max 100 words)",
  "constraints": ["Always call check_order first", "No refund without eligibility check"]
}
```

## Output Format

```python
{
  "system_prompt": "You are a customer order management agent for RetailBot...",
  "instructions": "...",
  "expected_output_format": {
    "fields": ["status_summary", "action_taken", "next_step"],
    "max_words": 100
  },
  "tool_usage_guidance": {
    "check_order": "Call FIRST for any order inquiry",
    "process_refund": "Call only if order is eligible",
    "escalate_ticket": "Call only if issue cannot be resolved automatically"
  }
}
```

## Error Handling

- **Agent ignores tool call order**: Zero-shot tool sequencing instructions
  must use imperative language: "You MUST call check_order BEFORE calling
  process_refund. There are no exceptions to this rule."
- **Output format not followed**: Add `output_schema` (Pydantic) to enforce
  format structurally rather than relying on text-only instructions.
- **Model asks clarifying questions when it should act**: Add: "Do not ask
  the user for information that is already available in session state or
  retrievable via tools."
- **Zero-shot fails on rare inputs**: Log failing cases and add them as
  few-shot examples. The transition from zero-shot to few-shot for specific
  edge cases is normal and expected.
- **Instruction length creep**: As constraints are added, the instruction
  grows. Set a hard limit of 1500 tokens. Beyond that, switch to few-shot
  or `InstructionProvider`.

## Examples

### Example 1 — Complete zero-shot instruction

```python
from google.adk.agents import LlmAgent
from pydantic import BaseModel
from my_tools import check_order, process_refund, escalate_ticket

class OrderResponse(BaseModel):
    status_summary: str        # One-sentence order status
    action_taken: str          # What the agent did
    next_step_for_user: str    # Clear next action for the user

ZERO_SHOT_INSTRUCTION = """
You are a customer order management specialist for RetailBot.

SCOPE: Order status checks, eligibility verification, refund processing, ticket escalation.
OUT OF SCOPE: Product recommendations, shipping logistics, billing disputes. Decline these politely.

TOOL PROTOCOL (follow this order strictly):
1. check_order(order_id): ALWAYS call this first. Provides order status, date, and eligibility.
2. process_refund(order_id, reason): Call ONLY if check_order returns status "delivered" or "damaged".
3. escalate_ticket(order_id, issue_summary): Call ONLY if process_refund fails or if the issue
   requires human judgment (fraud, complex disputes).

STATE INJECTION: The user's name is {user:name?} and their order ID is {order_id?}.

OUTPUT REQUIREMENTS:
- status_summary: One sentence, factual (e.g., "Your order ORD-789 was delivered on March 10.")
- action_taken: What you did (e.g., "I have initiated a refund for $45.00.")
- next_step_for_user: Clear guidance (e.g., "You will receive a refund in 3–5 business days.")

RULES:
- Never process an action without first calling check_order.
- Never invent order data — use only tool results.
- If the order ID is unknown, ask for it politely before calling any tool.
"""

agent = LlmAgent(
    name="order_management_agent",
    model="gemini-2.0-flash",
    instruction=ZERO_SHOT_INSTRUCTION,
    output_schema=OrderResponse,
    tools=[check_order, process_refund, escalate_ticket],
)
```

### Example 2 — Testing zero-shot sufficiency

```python
# Test matrix for zero-shot validation before moving to few-shot
TEST_CASES = [
    "My order hasn't arrived",             # → check_order → escalate_ticket
    "I want a refund for order ORD-456",   # → check_order → process_refund
    "What's the status of my order?",      # → check_order
    "Can you recommend a product?",        # → decline (out of scope)
    "Cancel my order",                     # → check_order → escalate_ticket
]
# If ≥ 3 of 5 fail → add few-shot examples for failing patterns
```

## Edge Cases

- **User provides no order ID**: Zero-shot should detect missing required
  info and ask. Add: "If the user hasn't provided an order_id, ask for it
  before calling any tool."
- **Tool returns an error**: Zero-shot agents need explicit handling guidance.
  Add: "If check_order returns an error, explain to the user and escalate_ticket."
- **Large context from state injection** (long order history injected): Inject
  only what's needed per turn. Use `{order_id?}` not `{full_order_history?}`.
- **Model adds unnecessary caveats** ("I'm an AI and cannot guarantee..."): Add:
  "Do not include AI disclaimers in your responses unless legally required."

## Integration Notes

- **`google-adk-few-shot-prompting`**: Zero-shot is the starting point; add
  few-shot examples when specific patterns consistently fail.
- **`LlmAgent.output_schema`**: The most reliable zero-shot output format
  enforcement — use Pydantic instead of text-only format instructions.
- **`google-adk-instruction-tuning`**: When zero-shot performance is
  suboptimal, instruction tuning refines the zero-shot prompt systematically.
- **Tool docstrings**: In zero-shot mode, these are critical — they provide
  the model its only signal about when to call each tool.
