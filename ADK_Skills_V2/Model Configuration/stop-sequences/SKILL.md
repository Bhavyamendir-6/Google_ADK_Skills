---
name: stop-sequences
description: >
  Use this skill when configuring stop sequences for a Google ADK LlmAgent to
  cause the model to halt generation at specific token patterns. Covers selecting
  appropriate stop sequences for structured output termination, turn-based
  conversations, tool call boundaries, and streaming responses; applying them
  via GenerateContentConfig; and validating that stop sequences work as expected.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: model-configuration
---

## Purpose

This skill configures `stop_sequences` for Google ADK `LlmAgent`. Stop sequences
are strings that cause the model to halt generation immediately when they appear
in the output, without including the stop sequence itself in the response. They
are used to: prevent runaway generation, enforce turn boundaries in structured
multi-turn conversations, terminate structured format blocks (e.g., JSON),
and limit output to specific sections of a longer potential generation.
Properly configured stop sequences are a lightweight, reliable way to control
output length and structure without changing temperature or top_k/top_p.

## When to Use

- When the agent's output must terminate at a specific pattern (e.g., end of
  a JSON block, end of a numbered list, turn separator).
- When the model generates content beyond the intended response boundary (e.g.,
  continues generating imaginary user messages after its own response).
- When implementing a structured multi-turn prompt format with explicit turn markers.
- When `max_output_tokens` alone doesn't precisely terminate at a semantic boundary.
- When limiting code or JSON generation to a single block.

## When NOT to Use

- Do not use stop sequences that may legitimately appear in valid outputs (e.g.,
  using `"\n\n"` as a stop sequence for a multi-paragraph response).
- Do not use stop sequences as the only length control mechanism — combine with
  `max_output_tokens`.
- Do not apply stop sequences for content moderation — they only affect when
  generation stops, not what can be generated.

## Google ADK Context

- **`GenerateContentConfig.stop_sequences`**: List of strings (up to 5). The
  model halts generation when any stop sequence is detected in the output. The
  stop sequence itself is excluded from the output.
- **Stop sequence patterns for ADK agents**:
  - `"\nUSER:"` — Stops before generating imaginary user turn.
  - `"\nAGENT:"` — Stops before generating a duplicate agent turn.
  - `"```"` — Stops after a JSON or code block closes.
  - `"---END---"` — Custom boundary marker injected in instruction.
- **Turn-based conversation stop**: Without stop sequences, some models continue
  generating imaginary dialogue after their response. Use `"\nUser:"` or `"\nHuman:"`.
- **Stop sequences in streaming**: The stream is cut when the stop sequence is
  detected. The partial generation up to (not including) the stop sequence is delivered.
- **Instruction alignment**: The stop sequence must align with patterns the
  model will actually generate. If the instruction uses `"USER:"` but the model
  generates `"User:"`, the stop won't trigger.

## Capabilities

- Halts model generation at specific string boundaries.
- Prevents imaginary turn generation in structured conversation prompts.
- Terminates code or JSON blocks at closing markers.
- Works in both streaming and non-streaming mode.
- Combines with `max_output_tokens` for dual-layer output control.
- Supports up to 5 stop sequences simultaneously.

## Step-by-Step Instructions

1. **Identify the termination pattern** — where the model must stop:
   - After responding in a structured prompt format → `"\nUSER:"` or `"\nHuman:"`
   - After a JSON block → `"\n```"` or `"}"` (careful — may appear mid-JSON)
   - After a numbered list → custom marker `"\n---DONE---"`

2. **Add the termination marker to the instruction** if it's a custom pattern:
   ```
   Respond to the user query. After your response, write exactly "---DONE---" on a new line.
   ```

3. **Configure stop sequences**:
   ```python
   from google.genai.types import GenerateContentConfig
   config = GenerateContentConfig(
       stop_sequences=["\n---DONE---", "\nUSER:"],
   )
   ```

4. **Apply to agent**:
   ```python
   agent = LlmAgent(
       name="my_agent",
       model="gemini-2.0-flash",
       generate_content_config=config,
   )
   ```

5. **Test stop sequence triggering** by running a query and verifying the output
   does NOT contain the stop sequence but IS properly terminated.

6. **Combine with `max_output_tokens`** for defense-in-depth:
   ```python
   config = GenerateContentConfig(
       stop_sequences=["\nUSER:", "\n---DONE---"],
       max_output_tokens=512,
   )
   ```

## Input Format

```python
{
  "model": "gemini-2.0-flash",
  "temperature": 0.3,
  "top_k": 40,
  "top_p": 0.95,
  "max_tokens": 512,
  "stop_sequences": ["\nUSER:", "---DONE---"],
  "stream": False,
  "response_schema": {},
  "deterministic": False
}
```

## Output Format

```python
{
  "model_used": "gemini-2.0-flash",
  "configuration_applied": {
    "stop_sequences": ["\nUSER:", "---DONE---"],
    "max_output_tokens": 512
  },
  "output_valid": True,
  "structured_output": {}
}
```

## Error Handling

- **Stop sequence never triggers** (model uses different casing/format):
  Inspect the raw model output via ADK Web UI Trace View to see what format
  the model produces, and match the stop sequence exactly.
- **Stop sequence triggers too early** (appears mid-response): Choose a stop
  sequence that is unlikely to appear naturally. Use `"---BOUNDARY---"` type
  markers injected by the instruction.
- **More than 5 stop sequences needed**: Combine multiple patterns into a single
  distinctive marker.
- **Stop sequence not respected in streaming**: Verify that the streaming event
  consumer correctly handles partial-stream cutoff events.
- **ADK does not expose stop reason**: Inspect output length — if it's
  significantly shorter than `max_output_tokens` and ends cleanly, a stop
  sequence likely triggered.

## Examples

### Example 1 — Turn-boundary stop for structured Q&A agent

```python
from google.adk.agents import LlmAgent
from google.genai.types import GenerateContentConfig

QA_CONFIG = GenerateContentConfig(
    temperature=0.3,
    max_output_tokens=400,
    stop_sequences=["\nUser:", "\nHuman:", "\nQuestion:"],  # Prevent imaginary turn generation
)

qa_agent = LlmAgent(
    name="qa_agent",
    model="gemini-2.0-flash",
    instruction="""
    You are a customer FAQ assistant. Answer questions directly and concisely.
    After your answer, stop. Do not generate additional questions or user dialogue.
    """,
    generate_content_config=QA_CONFIG,
)
```

### Example 2 — Custom stop marker for structured outputs

```python
from google.genai.types import GenerateContentConfig

STRUCTURED_CONFIG = GenerateContentConfig(
    temperature=0.1,
    max_output_tokens=800,
    stop_sequences=["---END REPORT---"],
)

report_agent = LlmAgent(
    name="report_agent",
    model="gemini-2.0-flash",
    instruction="""
    Generate a concise flight analysis report.
    Structure your report with:
    - SUMMARY: [one paragraph]
    - KEY FINDINGS: [3-5 bullet points]
    - RECOMMENDATION: [one sentence]

    After RECOMMENDATION, write exactly: ---END REPORT---
    """,
    generate_content_config=STRUCTURED_CONFIG,
)
```

### Example 3 — Code generation with stop at block boundary

```python
from google.genai.types import GenerateContentConfig

CODE_CONFIG = GenerateContentConfig(
    temperature=0.0,
    max_output_tokens=1024,
    stop_sequences=["\n```\n"],  # Stop after closing code fence
)

code_agent = LlmAgent(
    name="code_generator",
    model="gemini-2.0-flash",
    instruction="""
    Generate a Python function for the requested task.
    Output ONLY the code block wrapped in triple backticks.
    Do not add explanations before or after the code block.
    """,
    generate_content_config=CODE_CONFIG,
)
```

## Edge Cases

- **Stop sequence in the middle of a JSON value**: Using `"}"` or `"\n"` as a
  stop sequence breaks mid-JSON. Use full-block markers like `"```"` or custom
  end-of-JSON signals like `"---JSON_END---"` that the instruction tells the
  model to produce.
- **Long response without stop sequence**: The model generates valid content
  but never reaches the stop boundary. Always pair stop sequences with
  `max_output_tokens` as a fallback.
- **Stop sequence case-sensitivity**: Stop sequences are case-sensitive.
  `"USER:"` will not match `"User:"`. Test via Trace View with the actual model
  output before deploying.
- **Streaming and stop sequences**: The stream is cut immediately when the
  stop sequence is detected. The streaming consumer should handle abrupt cutoff.

## Integration Notes

- **`google-adk-token-limits`**: Stop sequences and `max_output_tokens` are
  complementary controls. Apply both together.
- **`google-adk-structured-json-output`**: Use stop sequences to terminate JSON
  block generation cleanly when not using `output_schema`.
- **`google-adk-streaming-responses`**: Stop sequences work identically in
  streaming mode. The stream is cut at the stop sequence in both modes.
- **`google-adk-output-formatting`**: Output format rules in the instruction
  define what patterns to use as stop sequences.
