---
name: top-k-sampling
description: >
  Use this skill when configuring the top_k sampling parameter for a Google
  ADK LlmAgent to control the vocabulary breadth of the model's token sampling.
  Covers selecting appropriate top_k values for focused vs. diverse outputs,
  applying top_k via GenerateContentConfig, and understanding how top_k interacts
  with temperature and top_p in ADK model inference.
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

This skill configures the `top_k` sampling parameter for Google ADK `LlmAgent`.
Top-k sampling restricts token selection to the `k` highest-probability tokens
at each generation step. Lower values of `k` (e.g., 1–10) produce more focused,
conservative outputs by limiting vocabulary; higher values (e.g., 40–100) allow
more diverse word choices. In combination with temperature and top_p, top_k is
part of the complete sampling configuration stack that controls output quality,
diversity, and consistency for ADK agents.

## When to Use

- When the agent produces unusual vocabulary or unexpected token choices despite
  low temperature — apply low top_k (1–10) to restrict vocabulary further.
- When the agent produces repetitive outputs because high temperature is limited
  by a low top_k — raise top_k (40–100) to allow more word variety.
- When configuring a structured output agent — use low top_k (1–20) to minimize
  format violations.
- When optimizing for a specific creativity-reliability tradeoff alongside
  temperature and top_p.

## When NOT to Use

- Do not tune top_k in isolation without testing it in combination with temperature
  and top_p — they interact and must be tuned as a group.
- Do not set top_k=1 (greedy) for open-ended generation — it produces extremely
  mechanical, repetitive responses.
- Do not apply top_k changes without running the evaluation suite to measure impact.

## Google ADK Context

- **`GenerateContentConfig.top_k`**: Integer 1–100. Restricts the sampling pool
  to the top `k` tokens by probability at each step.
- **Top-k semantics**:
  - `top_k=1` → Greedy decoding. Same as temperature=0 effect.
  - `top_k=5–10` → Very focused. Good for code, structured output, deterministic tools.
  - `top_k=20–40` → Default range. Balanced focus and variety.
  - `top_k=64–100` → Wide vocabulary. Good for creative diversity.
- **Interaction with temperature**: Temperature scales the probability distribution
  before top_k is applied. High temperature widens the distribution; top_k then
  caps how many tokens can be sampled from it.
- **Interaction with top_p**: Top-p additionally filters the top_k pool by
  cumulative probability. Both can be applied simultaneously.
- **Gemini defaults**: Gemini 2.0 models use `top_k=40` by default.
- **`generate_content_config`**: The field on `LlmAgent` where `top_k` is passed.

## Capabilities

- Restricts model vocabulary to top-k tokens per generation step.
- Controls focus vs. diversity of agent outputs.
- Reduces hallucinated or off-topic token choices with low top_k.
- Increases output word variety for creative tasks with high top_k.
- Combines with temperature and top_p for complete sampling configuration.

## Step-by-Step Instructions

1. **Determine the task type** and desired output focus:
   - Structured/tool-calling → `top_k=1–10`
   - Balanced explanation → `top_k=20–40`
   - Creative diverse output → `top_k=64–100`

2. **Import config class**:
   ```python
   from google.genai.types import GenerateContentConfig
   ```

3. **Apply top_k**:
   ```python
   config = GenerateContentConfig(
       temperature=0.1,
       top_k=10,
       top_p=0.95,
   )
   agent = LlmAgent(
       name="my_agent",
       model="gemini-2.0-flash",
       generate_content_config=config,
   )
   ```

4. **Run identical-input eval** (same query, 5 runs) and compare outputs.
   High variance = top_k too high. Mechanical/repetitive = top_k too low.

5. **Adjust** until the variance matches the expected level for the task.

## Input Format

```python
{
  "model": "gemini-2.0-flash",
  "temperature": 0.1,
  "top_k": 10,
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
  "configuration_applied": {"top_k": 10, "temperature": 0.1},
  "output_valid": True,
  "structured_output": {}
}
```

## Error Handling

- **`top_k` out of range** (< 1 or > 100): `GenerateContentConfig` raises `ValueError`.
  Validate before setting.
- **top_k=1 causes repetitive outputs** in long generation: Increase to 5–10.
  Full greedy decoding (`top_k=1`) can cause looping in longer text sequences.
- **top_k and temperature conflict** (low temp + high k or high temp + low k):
  Test empirically. High temperature + low top_k is unusual but valid. Low temp +
  high top_k has minimal practical effect (low temp already concentrates probability).
- **Model-specific defaults ignored**: Always explicitly set top_k in `GenerateContentConfig`
  for production agents — never rely on model defaults, which may change.

## Examples

### Example 1 — Focus configuration for structured tool calls

```python
from google.adk.agents import LlmAgent
from google.genai.types import GenerateContentConfig

PRECISE_CONFIG = GenerateContentConfig(
    temperature=0.05,
    top_k=5,      # Very restricted vocabulary — maximizes tool call precision
    top_p=0.9,
)

structured_agent = LlmAgent(
    name="data_extractor",
    model="gemini-2.0-flash",
    instruction="Extract product information and populate the output schema.",
    generate_content_config=PRECISE_CONFIG,
    output_schema=ProductSchema,
)
```

### Example 2 — Diversity configuration for creative tasks

```python
from google.genai.types import GenerateContentConfig

CREATIVE_CONFIG = GenerateContentConfig(
    temperature=1.0,
    top_k=64,     # Wide vocabulary for creative diversity
    top_p=0.95,
)

creative_agent = LlmAgent(
    name="creative_writer",
    model="gemini-2.0-flash",
    instruction="Generate creative product descriptions that vary in style.",
    generate_content_config=CREATIVE_CONFIG,
)
```

### Example 3 — A/B testing top_k values

```python
from google.genai.types import GenerateContentConfig

def create_agent_with_top_k(top_k_value: int):
    return LlmAgent(
        name=f"agent_topk_{top_k_value}",
        model="gemini-2.0-flash",
        instruction="Classify the customer intent from the message.",
        generate_content_config=GenerateContentConfig(temperature=0.1, top_k=top_k_value),
    )

# Test top_k values: 5, 20, 40
for k in [5, 20, 40]:
    agent = create_agent_with_top_k(k)
    # Run eval and compare tool_trajectory_avg_score
```

## Edge Cases

- **Very long outputs with low top_k**: May produce looping or stuttering text.
  Add `stop_sequences` to limit output length as a safety mechanism.
- **`top_k` combined with `output_schema`**: Top-k helps JSON format compliance.
  Use `top_k ≤ 20` with `output_schema` for best schema adherence.
- **Cross-language generation**: Low top_k may restrict vocabulary too much for
  non-English languages with larger token vocabularies. Use top_k ≥ 20 for
  multilingual agents.

## Integration Notes

- **`google-adk-temperature-tuning`**: Tune temperature first, then adjust top_k
  to further refine output focus or diversity.
- **`google-adk-top-p-sampling`**: Applied in the same `GenerateContentConfig`.
  Top-p provides cumulative probability filtering after top_k vocabulary restriction.
- **`google-adk-deterministic-output`**: For maximum determinism, use top_k=1
  (or top_k=5 with temperature=0) plus seed.
- **`google-adk-structured-json-output`**: Use low top_k (≤ 20) to improve
  structured JSON schema compliance.
