---
name: few-shot-prompting
description: >
  Use this skill when including concrete input-output examples directly within
  a Google ADK LlmAgent's instruction to guide behavior, enforce output patterns,
  and demonstrate correct tool usage. Covers embedding Q&A pairs, tool call
  examples, and structured output examples in the instruction field to reduce
  hallucination and improve output consistency.
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

This skill embeds concrete worked examples (few-shot demonstrations) into
Google ADK `LlmAgent` instructions to teach the model exactly what good behavior
looks like. Few-shot prompting is most effective when the agent needs to
replicate a specific output format, follow a specific tool call sequence, or
produce responses in a non-standard style that abstract instructions alone
cannot reliably produce. Examples anchor the model's behavior to concrete
patterns, dramatically improving output consistency.

## When to Use

- When abstract instructions alone are insufficient to produce the correct output
  format or tool call sequence.
- When the agent must produce highly specific output patterns (e.g., JSON, tables,
  numbered lists with specific field ordering).
- When the correct tool call sequence is complex and needs demonstration.
- When the agent serves a specialized domain where standard model behavior
  diverges from the required behavior.
- When `response_match_score` is high-variance and needs anchoring to a pattern.

## When NOT to Use

- Do not include examples that contain real user PII — use synthetic placeholders.
- Do not use more than 5 examples in the instruction — beyond that, token count
  inflates while marginal benefit decreases.
- Do not use few-shot examples as a substitute for fixing the underlying tool or
  data quality issues.

## Google ADK Context

- **`LlmAgent.instruction`**: Few-shot examples are embedded in the instruction
  string. They appear as `USER:` / `AGENT:` formatted example turns or as
  explicit `Example N:` sections.
- **Tool call demonstrations**: Show the correct tool call sequence using
  pseudocode or natural language in the examples.
- **`output_schema`**: When combined with structured output schemas, few-shot
  examples can demonstrate the expected JSON structure in the instruction.
- **Token budget**: Each example consumes tokens at every LLM call. Count tokens
  carefully. Use a maximum of 3–5 examples.
- **ADK Agent Execution Lifecycle**: Examples in the instruction are included
  in the system prompt at every invocation — they are always visible to the model.
- **Dynamic few-shot selection**: For large example libraries, use an
  `InstructionProvider` that selects relevant examples from `session.state`
  based on the current query type.

## Capabilities

- Embeds concrete input-output pairs in the `LlmAgent.instruction`.
- Demonstrates correct tool call sequences in example form.
- Anchors output formatting to concrete examples.
- Dynamically selects relevant examples via `InstructionProvider`.
- Improves output consistency for specialized domain tasks.
- Reduces hallucination by showing the model exactly what to produce.

## Step-by-Step Instructions

1. **Identify the behavior to anchor** — what specific output pattern or tool
   call sequence must the model replicate?

2. **Write 3–5 representative examples** covering:
   - Common cases the agent will encounter.
   - Edge cases that require specific handling.
   - Examples with and without tool calls.

3. **Format examples consistently**:
   ```
   EXAMPLE 1:
   USER: Find flights from JFK to LHR on March 15
   AGENT REASONING: The user wants to search for flights. I must call search_flights.
   AGENT ACTION: [calls search_flights(origin="JFK", destination="LHR", date="2024-03-15")]
   AGENT RESPONSE: "I found 3 flights from JFK to LHR on March 15..."
   ```

4. **Embed examples in the instruction** after the role and rules sections:
   ```python
   instruction = f"""
   {ROLE_SECTION}
   {RULES_SECTION}

   --- EXAMPLES ---
   {EXAMPLE_1}
   {EXAMPLE_2}
   {EXAMPLE_3}
   --- END EXAMPLES ---
   """
   ```

5. **Count tokens** in the instruction. Keep total under 2000 tokens for flash
   models or 4000 tokens for pro models.

6. **Test with inputs that match and diverge from examples** to verify
   generalization.

7. **Use dynamic example selection** for large example libraries:
   ```python
   def select_examples(ctx: ReadonlyContext) -> str:
       query_type = ctx.state.get("query_type", "general")
       return EXAMPLES_BY_TYPE.get(query_type, GENERAL_EXAMPLES)
   ```

## Input Format

```python
{
  "agent_goal": "Accurate flight search with correct tool sequence",
  "user_input": "Find flights from NYC to Paris next Monday",
  "context": {"query_type": "flight_search"},
  "tools": ["search_flights", "process_payment"],
  "output_format": "numbered list with airline, time, price",
  "constraints": ["Always call search_flights before showing prices"]
}
```

## Output Format

```python
{
  "system_prompt": "You are FlightBot...\n--- EXAMPLES ---\nEXAMPLE 1:\n...",
  "instructions": "...",
  "expected_output_format": {"type": "numbered_list", "fields": ["airline", "time", "price"]},
  "tool_usage_guidance": {
    "search_flights": "Demonstrated in Example 1",
    "process_payment": "Demonstrated in Example 2"
  }
}
```

## Error Handling

- **Examples contradict the rules section**: Audit for conflicts. Examples and
  rules must be consistent — examples take precedence in practice.
- **Model over-fits to examples** (copies example output verbatim): Vary the
  specifics of examples — different destinations, prices, dates.
- **Examples become outdated** (e.g., example prices are stale): Use abstract
  placeholders (`$[PRICE]`) rather than concrete values for time-sensitive data.
- **Too many examples inflate context**: Implement dynamic selection — use an
  `InstructionProvider` that injects only relevant examples based on query type.
- **Example format inconsistency**: Enforce a single format: `EXAMPLE N:` /
  `USER:` / `AGENT:`. Inconsistent formats confuse the model.
- **Missing edge case in examples**: Add the failing case as a new example.
  Few-shot examples are the fastest fix for rare, specific failure modes.

## Examples

### Example 1 — Instruction with embedded few-shot examples

```python
from google.adk.agents import LlmAgent

FEW_SHOT_INSTRUCTION = """
You are FlightBot, a flight booking assistant for AcmeAir.

RULES:
- Always call search_flights before quoting prices.
- Present results as a numbered list.
- Confirm before calling process_payment.

--- EXAMPLES ---

EXAMPLE 1 — Flight Search:
USER: "Find flights from London to New York on April 10"
AGENT THOUGHT: The user wants to search for flights. I must call search_flights first.
AGENT ACTION: search_flights(origin="LHR", destination="JFK", date="2024-04-10")
AGENT RESPONSE:
"Here are the available flights:
1. British Airways BA 178 — Departs 10:30 | Arrives 13:45 | 8h 15m | $520
2. Virgin Atlantic VS 4 — Departs 12:00 | Arrives 15:20 | 8h 20m | $490
Which flight would you like to book?"

EXAMPLE 2 — Payment Flow:
USER: "Book option 1 please"
AGENT THOUGHT: The user confirmed option 1 (BA 178). I now have a selection.
AGENT ACTION: (No tool call yet — ask for confirmation first)
AGENT RESPONSE:
"To confirm: British Airways BA 178 from LHR to JFK on April 10 for $520.
Shall I proceed with booking? Please reply 'yes' to confirm."

USER: "yes"
AGENT THOUGHT: The user confirmed. I can now call process_payment.
AGENT ACTION: process_payment(booking_id="BA178_20240410", card_token="[from state]")
AGENT RESPONSE:
"Your booking is confirmed! Booking reference: BOOK-92834."

EXAMPLE 3 — Off-topic:
USER: "Can you also book me a hotel?"
AGENT RESPONSE:
"I specialize in flight bookings only. I'm not able to book hotels. Can I help you with a flight?"

--- END EXAMPLES ---
"""

agent = LlmAgent(
    name="few_shot_flight_agent",
    model="gemini-2.0-flash",
    instruction=FEW_SHOT_INSTRUCTION,
    tools=[search_flights, process_payment],
)
```

### Example 2 — Dynamic example selection via InstructionProvider

```python
from google.adk.agents.readonly_context import ReadonlyContext

EXAMPLES = {
    "flight_search": """
EXAMPLE: Flight Search
USER: "Find flights from NYC to Paris"
AGENT: [calls search_flights(origin="JFK", destination="CDG", date="...")] → presents numbered list.
""",
    "booking": """
EXAMPLE: Booking Confirmation
USER: "Book option 2"
AGENT: Confirms selection, asks "yes" confirmation, then calls process_payment.
""",
    "refund": """
EXAMPLE: Refund Request  
USER: "Cancel my booking BOOK-123"
AGENT: [calls check_booking(booking_id="BOOK-123")] → presents cancellation terms → awaits confirmation.
""",
}

def dynamic_few_shot_provider(ctx: ReadonlyContext) -> str:
    query_type = ctx.state.get("query_type", "flight_search")
    example = EXAMPLES.get(query_type, EXAMPLES["flight_search"])
    return f"""
You are FlightBot for AcmeAir. Always call tools before quoting data.

--- RELEVANT EXAMPLE ---
{example}
--- END EXAMPLE ---
"""
```

## Edge Cases

- **User input matches example exactly** (model copies example output): Use
  abstract examples with placeholder values, not concrete values that match
  real queries.
- **Example demonstrates a deprecated tool**: Update examples when tools change.
  Stale examples teach the model to call tools that no longer exist.
- **Few-shot works in testing but fails in production** (distribution shift):
  Add examples from real production failure cases discovered via error analysis.
- **Multi-turn examples** showing a full conversation are the most effective
  but also the most token-expensive. Limit multi-turn examples to 1–2; use
  single-turn examples for the rest.

## Integration Notes

- **`google-adk-instruction-tuning`**: When abstract instructions fail and
  scores don't improve, switch to few-shot examples as part of the tuning process.
- **`google-adk-structured-prompting`**: Combine few-shot examples with
  structured output schemas to teach the model both format and content.
- **`InstructionProvider`**: Required for dynamic few-shot selection based on
  query type. Store example categories in `app:` scoped state for easy updates.
- **`google-adk-agent-evaluation-metrics`**: Run `tool_trajectory_avg_score`
  and `response_match_score` to measure whether example embedding improved alignment.
