---
name: model-selection-optimization
description: >
  Use this skill when selecting the optimal Gemini model variant for each
  component of a Google ADK agent pipeline to balance quality, latency, and cost.
  Covers comparing Flash-Lite, Flash, and Pro variants by capability, building
  a model routing layer, validating quality after downgrade, and configuring
  dynamic model selection based on query complexity at runtime.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: performance-optimization
---

## Purpose

This skill selects the optimal Gemini model variant for each component of a Google
ADK agent pipeline. Model selection is the highest-leverage performance optimization —
switching from `gemini-2.0-pro` to `gemini-2.0-flash` can cut cost by 10–15x with
minimal quality loss for many tasks. This skill provides a task-type → model mapping
framework, covers dynamic query-time model routing via a Flash-Lite classifier,
and specifies the quality validation process required before any model downgrade
is shipped to production.

## When to Use

- When cost or latency analysis reveals the current model is more powerful than
  the task requires.
- When different sub-agents in the same pipeline have different quality requirements.
- When implementing a routing layer to direct simple queries to Flash-Lite and
  complex queries to Flash or Pro.
- When validating whether a model downgrade is possible without quality regression.
- When selecting the initial model for a new ADK agent.

## When NOT to Use

- Do not downgrade the model without running an evaluation on representative queries.
- Do not use Flash-Lite for tasks requiring complex multi-step reasoning, long-form
  analysis, or nuanced instruction following.
- Do not use Pro for classification, extraction, or routing tasks — Flash-Lite
  handles these at a fraction of the cost.

## Google ADK Context

- **Model variants**:
  - `gemini-2.0-flash-lite`: Best for classification, routing, simple extraction,
    form-filling, short Q&A. Fastest and cheapest.
  - `gemini-2.0-flash`: Best for tool-use agents, structured output, multi-step
    reasoning, moderate analysis. Recommended default.
  - `gemini-2.0-pro`: Best for complex reasoning, nuanced analysis, long-form
    generation, advanced coding. Use sparingly.
- **`LlmAgent.model`**: Set to the model string for each sub-agent.
- **Dynamic model selection**: Use an `InstructionProvider` that reads a
  `query_complexity` state key (set by a Flash-Lite classifier) and routes to
  the appropriate model.
- **Model version pinning**: Use exact version strings (e.g., `gemini-2.0-flash-001`)
  for reproducibility. Model updates can change behavior.
- **Quality validation**: Run `AgentEvaluator` with `response_match_score` before
  and after model change. Require ≥ 95% of baseline score.

## Capabilities

- Selects the optimal model variant per sub-agent type.
- Implements dynamic query-time model routing via a Flash-Lite classifier.
- Validates quality with eval before deploying model downgrades.
- Pins model versions for reproducible production deployments.
- Configures model-level parameters aligned with each variant's capabilities.
- Measures per-model cost and latency for informed selection.

## Step-by-Step Instructions

1. **Map task types to optimal models**:
   | Task Type | Recommended Model |
   |---|---|
   | Classification, routing | `gemini-2.0-flash-lite` |
   | Extraction, summarization | `gemini-2.0-flash-lite` |
   | Tool-use, multi-step | `gemini-2.0-flash` |
   | Structured output | `gemini-2.0-flash` |
   | Complex analysis | `gemini-2.0-flash` |
   | Advanced reasoning | `gemini-2.0-pro` (only if Flash fails) |

2. **Assign models to sub-agents**:
   ```python
   classifier = LlmAgent(name="classifier", model="gemini-2.0-flash-lite", ...)
   worker = LlmAgent(name="worker", model="gemini-2.0-flash", ...)
   analyzer = LlmAgent(name="analyzer", model="gemini-2.0-flash", ...)  # Try Flash before Pro
   ```

3. **Implement dynamic routing** with a Flash-Lite classifier:
   ```python
   class QueryType(BaseModel):
       complexity: Literal["simple", "complex"]

   router = LlmAgent(
       name="router", model="gemini-2.0-flash-lite",
       instruction="Classify query as simple or complex based on reasoning depth required.",
       output_schema=QueryType, output_key="query_type",
   )
   ```

4. **Validate quality** after model downgrade:
   ```python
   await AgentEvaluator.evaluate(agent_module="my_agent", eval_dataset_file_path_or_dir="evals/")
   # Require: response_match_score >= 0.95 × baseline_score
   ```

5. **Pin model versions** for production stability:
   ```python
   FLASH_MODEL = "gemini-2.0-flash-001"  # Pinned version
   ```

6. **Monitor model performance** post-deployment using observability tools.

## Input Format

```python
{
  "operation": "model_routing",
  "execution_time_ms": 4200,
  "token_usage": 18000,
  "memory_usage_mb": 64,
  "cost_estimate": 0.0135,
  "retrieval_latency_ms": 0,
  "cache_available": False,
  "parallelizable": False,
  "context_size_tokens": 18000
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "model_downgrade: pro→flash + routing: flash-lite classifier",
  "performance_improvement": {
    "latency_reduction_ms": 1800,
    "token_reduction": 0,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.0112,
    "throughput_improvement": 2.1
  }
}
```

## Error Handling

- **Flash-Lite misclassifies complex queries**: Add evaluation examples for
  complex queries. Adjust the routing instruction: "When in doubt, classify as complex."
- **Flash produces wrong structured output** (Pro was needed): Temporarily revert
  to Pro for that sub-agent. Investigate if prompt changes can make Flash work.
- **Model version deprecated**: Update the version string. Run eval to validate.
- **Cost higher than expected after downgrade** (longer outputs from smaller model):
  Set `max_output_tokens` to limit verbosity.

## Examples

### Example 1 — Tiered model pipeline

```python
from pydantic import BaseModel
from typing import Literal
from google.adk.agents import LlmAgent, SequentialAgent
from google.genai.types import GenerateContentConfig

FLASH_LITE = "gemini-2.0-flash-lite"
FLASH = "gemini-2.0-flash"
FLASH_LITE_CONFIG = GenerateContentConfig(temperature=0, max_output_tokens=100)
FLASH_CONFIG = GenerateContentConfig(temperature=0.3, max_output_tokens=600)

class QueryComplexity(BaseModel):
    complexity: Literal["simple", "complex"]
    reason: str

# Stage 1: Flash-Lite routes the query
complexity_router = LlmAgent(
    name="complexity_router",
    model=FLASH_LITE,
    instruction=(
        "Assess the user query complexity.\n"
        "SIMPLE: single fact lookup, yes/no, short extraction, direct FAQ.\n"
        "COMPLEX: multi-step reasoning, comparisons, planning, long analysis."
    ),
    generate_content_config=FLASH_LITE_CONFIG,
    output_schema=QueryComplexity,
    output_key="query_complexity",
)

# Stage 2: Flash handles all responses (Flash-Lite for simple, Flash for complex)
# In practice, implement dynamic model selection based on output_key
responder = LlmAgent(
    name="intelligent_responder",
    model=FLASH,  # Default to Flash; upgrade to Pro only after eval shows need
    instruction="Query complexity: {query_complexity?}. Respond appropriately.",
    tools=[search_flights, get_policy, process_booking],
    generate_content_config=FLASH_CONFIG,
)

optimized_pipeline = SequentialAgent(
    name="model_optimized_pipeline",
    sub_agents=[complexity_router, responder],
)
```

## Edge Cases

- **Flash-Lite hallucination on domain-specific terminology**: Add few-shot examples
  to the Flash-Lite instruction. If hallucination persists, upgrade to Flash.
- **Model-specific output format differences**: Flash and Pro may produce slightly
  different structured output for the same schema. Run eval on each model variant.
- **Latency regression after model upgrade** (Flash→Pro for quality): Profile
  whether the Pro latency is acceptable. If not, optimize the Flash prompt instead.

## Integration Notes

- **`google-adk-cost-optimization`**: Model selection is the highest-impact
  cost optimization. Flash-Lite vs. Pro is a 10–15x cost difference.
- **`google-adk-latency-optimization`**: Flash-Lite is 2–3x faster than Flash.
  Model downgrade directly reduces inference latency.
- **`google-adk-prompt-optimization`**: A better prompt can often compensate
  for a model downgrade — try prompt optimization before model upgrade.
- **`google-adk-automated-evaluation`**: Required to validate quality before
  any model selection change ships to production.
