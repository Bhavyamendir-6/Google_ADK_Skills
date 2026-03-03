---
name: temperature-tuning
description: >
  Use this skill when configuring the temperature parameter for a Google ADK
  LlmAgent's model inference to control output randomness and creativity.
  Covers selecting appropriate temperature values for deterministic tool calls,
  creative generation, and structured output; applying temperature via
  generate_content_config; and validating that temperature settings produce
  the desired output consistency.
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

This skill configures the `temperature` inference parameter for Google ADK
`LlmAgent`. Temperature controls the randomness of the model's token sampling —
lower values (near 0.0) produce deterministic, focused outputs; higher values
(near 2.0) produce creative, diverse outputs. Proper temperature tuning directly
affects agent reliability: too high causes tool call inconsistency and output
drift; too low causes repetitive outputs and loss of nuance. This skill selects
the right temperature for each agent's specific task and configures it via
ADK's `generate_content_config`.

## When to Use

- When an agent produces inconsistent tool call sequences due to high-variance
  sampling — apply low temperature (≤ 0.1).
- When an agent produces repetitive, overly rigid outputs — apply moderate
  temperature (0.3–0.7).
- When the agent performs creative tasks (story generation, brainstorming) —
  apply higher temperature (0.7–1.2).
- When configuring a new `LlmAgent` and the default temperature is unknown
  or not set.
- When running structured output agents — always apply temperature ≤ 0.2.

## When NOT to Use

- Do not set temperature to 0.0 for all agents by default — it removes any
  flexibility in the model's response style and can cause looping behaviors.
- Do not use high temperature (> 1.0) for agents that call financial, medical,
  or safety-critical tools — high temperature increases hallucination risk.
- Do not tune temperature in isolation — combine with top_k and top_p tuning
  for complete sampling configuration.

## Google ADK Context

- **`generate_content_config`**: The ADK mechanism for passing inference
  parameters to the model. Applied on `LlmAgent` via the `generate_content_config`
  parameter, which accepts a `google.genai.types.GenerateContentConfig` object.
- **`GenerateContentConfig.temperature`**: Float value 0.0–2.0. Default varies
  by model. Gemini 2.0 models default to 1.0.
- **Temperature semantics**:
  - `0.0` → Greedy decoding. Most probable token always selected. Fully deterministic.
  - `0.1–0.3` → Near-deterministic. Good for tool calls, structured output.
  - `0.4–0.7` → Balanced. Good for explanation, summarization, customer support.
  - `0.8–1.2` → Creative. Good for content generation, brainstorming.
  - `1.3–2.0` → Highly random. Good for creative writing, diversity sampling.
- **Tool call consistency**: Temperature > 0.5 can cause the model to vary which
  tool it calls on repeated identical inputs. Use ≤ 0.1 for deterministic tool
  call behavior.
- **Temperature and `output_schema`**: For structured output agents, always use
  temperature ≤ 0.2 to minimize JSON format violations.
- **Temperature ≠ Temperature=0 always**: At temperature=0, the model may still
  produce slightly different outputs due to floating-point non-determinism in
  parallel GPU computation. Use `seed` for full reproducibility.

## Capabilities

- Configures temperature via `GenerateContentConfig` on `LlmAgent`.
- Reduces output variance for tool-calling and structured output agents.
- Increases output diversity for creative generation agents.
- Applies per-agent temperature in multi-agent pipelines.
- Validates temperature's effect on tool call consistency via evaluation.
- Combines temperature with top_k and top_p for complete sampling control.

## Step-by-Step Instructions

1. **Identify the agent's task type** and required output consistency:
   - Tool-calling, code generation, structured output → temperature ≤ 0.1
   - Summarization, Q&A, customer support → temperature 0.3–0.5
   - Creative writing, brainstorming → temperature 0.7–1.2

2. **Import the configuration class**:
   ```python
   from google.genai.types import GenerateContentConfig
   ```

3. **Create the config with temperature**:
   ```python
   config = GenerateContentConfig(temperature=0.1)
   ```

4. **Apply on `LlmAgent`**:
   ```python
   from google.adk.agents import LlmAgent
   agent = LlmAgent(
       name="my_agent",
       model="gemini-2.0-flash",
       instruction="...",
       generate_content_config=config,
   )
   ```

5. **Validate** by running the agent with the same input 3–5 times and checking
   output variance. If high variance persists, lower temperature further.

6. **Test tool call consistency** by running the eval suite with
   `tool_trajectory_avg_score`. Low scores under repeated execution indicate
   temperature is too high.

7. **Adjust** based on results. Document the chosen temperature in a comment.

## Input Format

```python
{
  "model": "gemini-2.0-flash",
  "temperature": 0.1,
  "top_k": 40,
  "top_p": 0.95,
  "max_tokens": 1024,
  "stop_sequences": [],
  "stream": False,
  "response_schema": {},
  "deterministic": False
}
```

## Output Format

```python
{
  "model_used": "gemini-2.0-flash",
  "configuration_applied": {"temperature": 0.1},
  "output_valid": True,
  "structured_output": {}
}
```

## Error Handling

- **Temperature out of range**: `GenerateContentConfig` raises `ValueError` if
  temperature < 0 or > 2. Always validate temperature values programmatically.
- **High temperature causes tool call failure**: If `tool_trajectory_avg_score`
  drops under repeated invocation, lower temperature to ≤ 0.1.
- **Temperature=0 still non-deterministic**: Add `seed=42` to `GenerateContentConfig`
  in addition to `temperature=0.0` for maximum reproducibility.
- **Creative task needs variety but structured format**: Use moderate temperature
  (0.5–0.7) with `output_schema` to balance creativity and format compliance.
- **Model ignores temperature setting**: Some model endpoints apply minimum
  temperature floors. Verify the Gemini API documentation for the specific
  model version.

## Examples

### Example 1 — Tool-calling agent with low temperature

```python
from google.adk.agents import LlmAgent
from google.genai.types import GenerateContentConfig
from my_tools import search_flights, process_payment

# Low temperature for consistent tool call sequences
TOOL_AGENT_CONFIG = GenerateContentConfig(
    temperature=0.05,  # Near-deterministic for reliable tool calls
)

flight_agent = LlmAgent(
    name="flight_booking_agent",
    model="gemini-2.0-flash",
    instruction="""
    You are a flight booking assistant. Call search_flights first, then
    process_payment only after user confirmation.
    """,
    generate_content_config=TOOL_AGENT_CONFIG,
    tools=[search_flights, process_payment],
)
```

### Example 2 — Creative agent with higher temperature

```python
from google.adk.agents import LlmAgent
from google.genai.types import GenerateContentConfig

# Higher temperature for creative diversity
CREATIVE_CONFIG = GenerateContentConfig(
    temperature=0.9,
    top_p=0.95,
    top_k=64,
)

creative_agent = LlmAgent(
    name="content_generator",
    model="gemini-2.0-flash",
    instruction="""
    You are a creative content writer. Generate engaging, unique marketing copy
    for products. Vary your style and vocabulary across responses.
    """,
    generate_content_config=CREATIVE_CONFIG,
)
```

### Example 3 — Per-pipeline-stage temperature configuration

```python
from google.adk.agents import LlmAgent, SequentialAgent
from google.genai.types import GenerateContentConfig

# Retrieval: near-deterministic
search_agent = LlmAgent(
    name="searcher",
    model="gemini-2.0-flash",
    generate_content_config=GenerateContentConfig(temperature=0.0),
    instruction="Search for products and return structured results.",
    tools=[search_catalog],
)

# Analysis: moderate temperature
analyst_agent = LlmAgent(
    name="analyst",
    model="gemini-2.0-pro",
    generate_content_config=GenerateContentConfig(temperature=0.3),
    instruction="Analyze search results and provide actionable insights.",
)

# Copy generation: creative temperature
writer_agent = LlmAgent(
    name="writer",
    model="gemini-2.0-flash",
    generate_content_config=GenerateContentConfig(temperature=1.0),
    instruction="Write compelling marketing copy based on the analysis.",
)

pipeline = SequentialAgent(
    name="content_pipeline",
    sub_agents=[search_agent, analyst_agent, writer_agent],
)
```

## Edge Cases

- **Structured output + high temperature**: Increases JSON schema violation rate.
  Never use temperature > 0.3 with `output_schema`.
- **Streaming + temperature**: Temperature applies per-token during streaming.
  No special handling needed — behavior is the same.
- **Very long outputs diverge** with moderate temperature: High temperature
  causes outputs to drift far from the instruction for long generations. Use
  low temperature (< 0.3) for long-form structured outputs.
- **Identical inputs produce identical outputs at temperature=0**: Expected behavior.
  This is the use case for temperature=0 in testing and evaluation.

## Integration Notes

- **`google-adk-deterministic-output`**: Combines temperature=0 with seed for
  full output reproducibility.
- **`google-adk-top-k-sampling`** and **`google-adk-top-p-sampling`**: Applied
  alongside temperature in the same `GenerateContentConfig`. All three interact.
- **`google-adk-structured-json-output`**: Always use temperature ≤ 0.2 when
  `output_schema` or `response_mime_type="application/json"` is set.
- **`google-adk-automated-evaluation`**: Run `tool_trajectory_avg_score` at
  different temperatures to find the optimal setting for each agent.
