---
name: top-p-sampling
description: >
  Use this skill when configuring the top_p (nucleus sampling) parameter for a
  Google ADK LlmAgent to control token sampling by cumulative probability
  threshold. Covers selecting appropriate top_p values for focused vs. diverse
  output, applying top_p via GenerateContentConfig, and understanding how top_p
  interacts with temperature and top_k in ADK model inference.
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

This skill configures the `top_p` (nucleus sampling) parameter for Google ADK
`LlmAgent`. Top-p sampling dynamically restricts token selection to the smallest
set of tokens whose cumulative probability mass exceeds the threshold `p`. Unlike
top_k (which is a fixed count), top_p adapts to the probability distribution —
using more tokens when the model is uncertain and fewer when it is confident.
This produces more natural, contextually appropriate diversity than fixed-k sampling,
and is the recommended primary diversity control for most ADK agent configurations.

## When to Use

- As the primary diversity control parameter alongside temperature (preferred
  over top_k alone for most tasks).
- When token selection should adapt dynamically to the model's confidence level.
- When balancing output quality and diversity across variable-difficulty queries.
- When configuring a structured output agent — use top_p ≤ 0.9 to improve
  JSON schema compliance.
- When the agent produces occasional unexpected token choices that top_k alone
  cannot address.

## When NOT to Use

- Do not lower top_p below 0.7 for creative generation tasks — it restricts
  vocabulary too much for nuanced writing.
- Do not use extreme top_p values (< 0.5) without testing empirically — outputs
  may become repetitive.
- Do not tune top_p without also reviewing temperature — they interact and must
  be considered together.

## Google ADK Context

- **`GenerateContentConfig.top_p`**: Float 0.0–1.0. Nucleus sampling threshold.
  Only tokens whose cumulative probability ≥ 1-p are excluded from sampling.
- **Top-p semantics**:
  - `top_p=1.0` → No nucleus filtering. Full vocabulary available.
  - `top_p=0.95` → 95% probability mass used. Default for most tasks.
  - `top_p=0.85–0.90` → Slightly more conservative.
  - `top_p=0.70–0.80` → Focused. Good for structured/factual generation.
  - `top_p < 0.70` → Very restricted. Risk of repetition.
- **Interaction with temperature**: Temperature is applied first (scales logits),
  then top_p filters by cumulative probability on the scaled distribution.
- **Interaction with top_k**: top_k restricts to top-k tokens; top_p then
  further restricts to the nucleus probability mass. Both can be applied together.
- **Gemini defaults**: `top_p=0.95` is the Gemini 2.0 default.
- **`generate_content_config`**: Applied to `LlmAgent` via this parameter.

## Capabilities

- Controls token sampling breadth via cumulative probability threshold.
- Adapts vocabulary size dynamically to model confidence at each step.
- Improves output quality by excluding low-probability tail tokens.
- Reduces hallucination risk through conservative probability thresholds.
- Combines with temperature and top_k for complete sampling stack configuration.

## Step-by-Step Instructions

1. **Determine task type** and required output predictability:
   - Structured/tool/JSON → `top_p=0.80–0.90`
   - Balanced Q&A/support → `top_p=0.92–0.95`
   - Creative/diverse → `top_p=0.95–1.00`

2. **Create config with top_p**:
   ```python
   from google.genai.types import GenerateContentConfig
   config = GenerateContentConfig(
       temperature=0.3,
       top_k=40,
       top_p=0.95,
   )
   ```

3. **Apply to agent**:
   ```python
   agent = LlmAgent(
       name="my_agent",
       model="gemini-2.0-flash",
       generate_content_config=config,
   )
   ```

4. **Test diversity and quality** by running 10 identical queries. Adjust top_p
   up if outputs are repetitive; adjust down if outputs drift from expected format.

5. **Run eval** (`response_match_score`) to confirm quality with new top_p.

## Input Format

```python
{
  "model": "gemini-2.0-flash",
  "temperature": 0.3,
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
  "configuration_applied": {"top_p": 0.95, "temperature": 0.3, "top_k": 40},
  "output_valid": True,
  "structured_output": {}
}
```

## Error Handling

- **`top_p` out of range** (< 0 or > 1): `GenerateContentConfig` raises `ValueError`.
  Validate before applying.
- **`top_p=1.0` with high temperature** produces near-random outputs: Reduce
  temperature or lower top_p to 0.95.
- **`top_p` very low causes repetition**: If the same phrase repeats across
  multiple generations, top_p is too low. Increase by 0.10 increments.
- **Conflict between top_p and response_schema compliance**: Low top_p (< 0.80)
  with complex schemas may restrict the model from producing valid JSON field names.
  Use `top_p=0.85` minimum for schemas with diverse field names.

## Examples

### Example 1 — Standard balanced configuration

```python
from google.adk.agents import LlmAgent
from google.genai.types import GenerateContentConfig

BALANCED_CONFIG = GenerateContentConfig(
    temperature=0.4,
    top_k=40,
    top_p=0.95,   # Standard nucleus sampling — most tasks
)

support_agent = LlmAgent(
    name="customer_support_agent",
    model="gemini-2.0-flash",
    instruction="Help customers with order inquiries professionally.",
    generate_content_config=BALANCED_CONFIG,
    tools=[check_order, process_refund],
)
```

### Example 2 — Conservative configuration for structured output

```python
from google.genai.types import GenerateContentConfig
from pydantic import BaseModel

class ProductData(BaseModel):
    name: str
    price: float
    category: str

STRUCTURED_CONFIG = GenerateContentConfig(
    temperature=0.1,
    top_k=20,
    top_p=0.85,   # Conservative nucleus for JSON schema compliance
)

extractor = LlmAgent(
    name="product_extractor",
    model="gemini-2.0-flash",
    instruction="Extract product data from the listing text.",
    generate_content_config=STRUCTURED_CONFIG,
    output_schema=ProductData,
)
```

### Example 3 — Testing top_p impact on output diversity

```python
from google.genai.types import GenerateContentConfig

def measure_output_diversity(agent, query: str, n_runs: int = 10) -> float:
    """Runs the agent n times and measures unique word count variance."""
    responses = []
    for _ in range(n_runs):
        # ... run agent and collect response text ...
        pass
    unique_words = len(set(w for r in responses for w in r.split()))
    return unique_words / n_runs  # diversity score

# Test: top_p=0.70 vs 0.95 diversity
for p in [0.70, 0.85, 0.95]:
    config = GenerateContentConfig(temperature=0.5, top_p=p)
    # measure_output_diversity(agent_with_config, "Write a product tagline")
```

## Edge Cases

- **Top_p interaction with very long outputs**: Top_p filters apply per token.
  Long outputs with low top_p may exhibit stronger repetition. Add `stop_sequences`
  to limit output length.
- **Multilingual agents**: Tokenization differs across languages. Top_p=0.90 may
  be too restrictive for languages with large token vocabularies. Test empirically.
- **top_p=0.0**: Not recommended. Produces degenerate outputs. Use temperature=0
  and top_k=1 for greedy decoding instead.

## Integration Notes

- **`google-adk-temperature-tuning`**: Primary control. Tune temperature first,
  then fine-tune with top_p.
- **`google-adk-top-k-sampling`**: Complementary control. top_k restricts vocab;
  top_p further filters by probability mass. Apply both together for precise control.
- **`google-adk-structured-json-output`**: Use top_p=0.85–0.90 for structured
  output tasks to maximize JSON schema adherence.
- **`google-adk-deterministic-output`**: Set top_p=1.0 (disable nucleus filtering)
  and use temperature=0 + seed for determinstic output (top_k=1 effect dominates).
