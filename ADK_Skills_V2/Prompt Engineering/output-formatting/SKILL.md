---
name: output-formatting
description: >
  Use this skill when controlling the visual structure, length, and presentation
  format of a Google ADK LlmAgent's text responses. Covers defining output format
  directives in the instruction (markdown, numbered lists, tables, JSON, plain text),
  using LlmAgent.output_schema for structured output, enforcing response length
  limits, and validating output format compliance in agent callbacks.
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

This skill governs how Google ADK `LlmAgent` formats its text output — the
visual structure, length, and presentation style of responses. Output formatting
directly affects user experience, downstream parseability, and consistency.
This skill covers adding format directives to the `instruction` field,
using `output_schema` for machine-parseable output, enforcing length limits,
and using `after_agent_callback` to validate or sanitize output format at runtime.

## When to Use

- When the agent's responses need a specific visual structure (numbered list,
  table, bullet points, markdown headers).
- When response length must be constrained (user-facing chat vs. report generation).
- When the agent's output is consumed by a downstream system and must be in a
  specific text format.
- When the agent produces inconsistent formatting that degrades user experience.
- When different response types (search results, summaries, confirmations) need
  distinct formats.

## When NOT to Use

- Do not specify formatting in the instruction for machine-parseable output —
  use `output_schema` instead.
- Do not over-specify formatting to the point of rigidity — leave the agent room
  to adapt structure to content.
- Do not apply rich markdown formatting for plain-text channels that don't render
  markdown (SMS, plain email).

## Google ADK Context

- **`LlmAgent.instruction`**: The primary mechanism for text-based output format
  directives. Clear, specific format instructions are required.
- **`LlmAgent.output_schema`**: For structured/machine-parseable output.
  Overrides text format instructions — the model outputs JSON conforming to
  the Pydantic schema.
- **`output_key`**: When `output_schema` is set, the parsed structured output
  is stored in `session.state[output_key]`.
- **`after_agent_callback`**: Can intercept and sanitize the final response text
  before it is returned — useful for stripping unwanted markdown, enforcing
  length, or reformatting.
- **ADK Agent Execution Lifecycle**: Output format is applied during the LLM
  call but can be post-processed in `after_agent_callback`.
- **Multi-agent pipelines**: Each sub-agent's output format is independent.
  The root agent's format is what the user sees.

## Capabilities

- Defines response length limits in the instruction field.
- Specifies list format (numbered, bulleted), table format, or markdown headers.
- Enforces consistent section structure in multi-part responses.
- Uses `output_schema` for fully structured JSON output.
- Implements `after_agent_callback` for output post-processing and validation.
- Applies different formats for different response types via `InstructionProvider`.
- Strips unwanted formatting for plain-text delivery channels.

## Step-by-Step Instructions

1. **Identify the target format** for each response type:
   - Search results → numbered list
   - Summaries → bullet points under headers
   - Confirmations → single paragraph, max 50 words
   - Machine output → Pydantic schema

2. **Write explicit format directives** in the instruction:
   ```
   OUTPUT FORMAT RULES:
   - Flight search results: Always use a numbered list with fields:
     "[N]. [Airline] [Flight#] — [Departs] | [Arrives] | [Duration] | $[Price]"
   - Booking confirmations: Single paragraph, max 50 words.
   - Error messages: Start with "I'm sorry, " followed by the issue and next step.
   - Off-topic responses: Exactly: "I specialize in [topic]. Is there a [topic] I can help with?"
   ```

3. **Set global length limits**:
   ```
   All responses must be under 150 words. If more detail is needed, offer to elaborate.
   ```

4. **Use `output_schema`** for structured output (see `structured-prompting` skill).

5. **Implement `after_agent_callback`** to validate format:
   ```python
   def enforce_format(callback_context: CallbackContext, response: Content) -> Optional[Content]:
       text = response.parts[0].text if response and response.parts else ""
       if len(text.split()) > 200:
           # Truncate with ellipsis
           truncated = " ".join(text.split()[:200]) + "..."
           from google.genai.types import Part
           return Content(parts=[Part(text=truncated)])
       return None
   ```

6. **Test format compliance** by running eval cases and checking that output
   matches the expected format template.

## Input Format

```python
{
  "agent_goal": "Present flight search results in consistent numbered list format",
  "user_input": "Find flights from NYC to London",
  "context": {"response_type": "flight_search_results"},
  "tools": ["search_flights"],
  "output_format": "numbered list: [N]. [Airline] [Flight#] — [Details] | $[Price]",
  "constraints": ["Max 150 words", "Must include all 5 fields per item"]
}
```

## Output Format

```python
{
  "system_prompt": "...\nOUTPUT FORMAT RULES:\n- Flight search results: numbered list...",
  "instructions": "...",
  "expected_output_format": {
    "search_results": "numbered_list",
    "confirmations": "single_paragraph_50w",
    "errors": "starts_with_I_m_sorry"
  },
  "tool_usage_guidance": {}
}
```

## Error Handling

- **Agent ignores format instructions**: Move format instructions to a clearly
  labeled section: `--- OUTPUT FORMAT RULES (MANDATORY) ---`. Models respond
  better to section headers.
- **Format varies across turns**: Add to instruction: "Apply the same output
  format in every response, regardless of the topic or complexity."
- **Markdown rendered as raw text in plain-text channel**: Use `after_agent_callback`
  to strip markdown using a regex: `re.sub(r'\*\*|\*|__|_|#{1,6} ', '', text)`.
- **Response too long**: Add a hard limit: "If your response would exceed 150
  words, summarize and offer 'Would you like more detail?'"
- **Structured output with extra narrative text**: Use `output_schema` to
  structurally enforce JSON — text format instructions alone are unreliable for
  strict JSON compliance.

## Examples

### Example 1 — Multi-format instruction with response-type detection

```python
from google.adk.agents import LlmAgent
from my_tools import search_flights, process_payment

FORMATTING_INSTRUCTION = """
You are FlightBot for AcmeAir. Adapt your output format to the response type:

FLIGHT SEARCH RESULTS:
Present as a numbered list. Each item must follow this exact template:
"[N]. [Airline] [Flight#] | Departs [TIME] → Arrives [TIME] | [DURATION] | $[PRICE]"
Example:
"1. British Airways BA 178 | Departs 10:30 → Arrives 13:45 | 8h 15m | $520"
Maximum 5 results. If more than 5, show top 5 by price.

BOOKING CONFIRMATION:
Single paragraph, max 60 words. Include: airline, flight number, date, route, price, booking reference.

ERROR / NOT FOUND:
Start with: "I'm sorry, " — then the issue — then suggest an alternative action.

OFF-TOPIC DECLINE:
Exactly: "I specialize in flight bookings. Can I help you find or book a flight?"

GLOBAL RULES:
- No markdown headers (##) in responses — plain text only.
- Never exceed 200 words in any response.
- Do not add caveats like "As an AI..." or "Please note that...".
"""

agent = LlmAgent(
    name="formatted_flight_agent",
    model="gemini-2.0-flash",
    instruction=FORMATTING_INSTRUCTION,
    tools=[search_flights, process_payment],
)
```

### Example 2 — after_agent_callback for output post-processing

```python
import re
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content, Part

MAX_WORDS = 200

def sanitize_output(callback_context: CallbackContext, response: Optional[Content]) -> Optional[Content]:
    """Enforces word limit and strips unwanted markdown from agent responses."""
    if not response or not response.parts:
        return response

    text = " ".join(p.text for p in response.parts if hasattr(p, "text") and p.text)
    if not text:
        return response

    # Strip markdown bold/italic (for plain-text channels)
    clean_text = re.sub(r'\*{1,2}|_{1,2}', '', text)

    # Enforce word limit
    words = clean_text.split()
    if len(words) > MAX_WORDS:
        clean_text = " ".join(words[:MAX_WORDS]) + "..."

    return Content(parts=[Part(text=clean_text.strip())])


agent = LlmAgent(
    name="plain_text_agent",
    model="gemini-2.0-flash",
    instruction="...",
    after_agent_callback=sanitize_output,
    tools=[search_flights],
)
```

### Example 3 — Dynamic format selection via InstructionProvider

```python
from google.adk.agents.readonly_context import ReadonlyContext

FORMATS = {
    "chat": "Conversational, max 100 words. No bullet points.",
    "report": "Use markdown headers (##), bullet points for details. Max 500 words.",
    "api": "Return only JSON. No explanatory text.",
}

def format_provider(ctx: ReadonlyContext) -> str:
    channel = ctx.state.get("meta:channel", "chat")
    fmt = FORMATS.get(channel, FORMATS["chat"])
    return f"""
You are a data analysis assistant.
OUTPUT FORMAT for this channel ({channel}): {fmt}
Always adhere to this format regardless of the complexity of the response.
"""
```

## Edge Cases

- **Agent outputs in a mix of formats across turns**: Add: "Maintain the same
  output format in every response during this session."
- **Format breaks with very short answers**: Add: "For very short answers
  (< 20 words), a single sentence is acceptable without the numbered list format."
- **Table format for non-tabular data**: Don't force table format on data that
  isn't naturally tabular. The model may produce poorly aligned tables.
- **Long URLs in formatted output**: Long URLs break numbered list formatting.
  Instruct: "For URLs, use this format: [Link text](url)" or omit URLs entirely.

## Integration Notes

- **`google-adk-structured-prompting`**: Machine-parseable output → use
  `output_schema`. Human-readable output formatting → use this skill.
- **`google-adk-system-prompts`**: Output format rules are a section within
  the full system prompt constructed by `system-prompts` skill.
- **`after_agent_callback`**: The runtime enforcement layer for output format.
  Use for sanitization, length enforcement, and channel-specific transformations.
- **`google-adk-dynamic-prompt-generation`**: When the output format must vary
  per session state, use `InstructionProvider` from that skill.
