---
name: deterministic-output
description: >
  Use this skill when a Google ADK LlmAgent must produce identical output for
  identical inputs across multiple invocations. Covers configuring temperature=0,
  seed, top_k=1, and top_p=1.0 in GenerateContentConfig to maximize output
  reproducibility; validating determinism in evaluation; and understanding the
  limits of determinism in GPU-based LLM inference.
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

This skill configures Google ADK `LlmAgent` for maximum output determinism —
producing as close to identical outputs as possible for identical inputs across
multiple invocations. Deterministic output is essential for reproducible
evaluation, regression testing, and scenarios where agents must produce consistent
decisions rather than varying responses. This skill covers setting temperature,
seed, top_k, and top_p in `GenerateContentConfig` to minimize stochasticity
and explains the practical limits of "true" determinism in Gemini inference.

## When to Use

- When running regression tests that compare exact agent output across code
  changes — variance makes tests flaky.
- When the agent performs deterministic classification or decision tasks where
  varying output is a bug.
- When a structured output agent must always produce the same extracted data
  from the same input.
- When debugging agent behavior — you need to reproduce an exact failure case.
- When evaluation requires identical outputs for methodological validity.

## When NOT to Use

- Do not enforce determinism for creative generation tasks — it defeats the purpose.
- Do not rely on `temperature=0` alone for guaranteed identical outputs — GPU
  floating-point non-determinism can still cause slight variations. Use `seed` too.
- Do not use deterministic configuration in production user-facing agents where
  response variety is desirable — users will notice the robotic repetition.

## Google ADK Context

- **`GenerateContentConfig.temperature`**: Set to `0.0` for greedy (most probable
  token) decoding — primary determinism control.
- **`GenerateContentConfig.seed`**: Integer seed for the random number generator.
  When `temperature=0` and `seed` is fixed, output is maximally reproducible
  within the same model version.
- **`GenerateContentConfig.top_k`**: Set to `1` for greedy decoding to reinforce
  temperature=0 effect.
- **`GenerateContentConfig.top_p`**: Set to `1.0` (no nucleus filtering) when
  temperature=0 is used to avoid any additional stochastic filtering.
- **Practical limits of determinism in Gemini**:
  - Parallel GPU computation introduces floating-point rounding differences.
  - Same `seed` + `temperature=0` produces nearly identical but not always 100%
    identical outputs.
  - For 100% reproducibility in testing, use mocked LLM responses.
- **ADK evaluation reproducibility**: Use `seed` in eval runs to minimize flaky
  test results from model variance.

## Capabilities

- Configures temperature, seed, top_k, top_p for maximum determinism.
- Reduces output variance for classification and decision-making agents.
- Produces consistent structured output across repeated invocations.
- Enables reproducible automated evaluation runs.
- Documents the expected determinism level and its limits.

## Step-by-Step Instructions

1. **Configure deterministic sampling parameters**:
   ```python
   from google.genai.types import GenerateContentConfig
   DETERMINISTIC_CONFIG = GenerateContentConfig(
       temperature=0.0,
       top_k=1,
       top_p=1.0,     # Disable nucleus filtering — greedy decoding takes over
       seed=42,       # Fixed seed for reproducibility
   )
   ```

2. **Apply to the agent**:
   ```python
   from google.adk.agents import LlmAgent
   agent = LlmAgent(
       name="deterministic_classifier",
       model="gemini-2.0-flash",
       instruction="Classify the customer intent as exactly one of: refund, status, complaint, other.",
       generate_content_config=DETERMINISTIC_CONFIG,
   )
   ```

3. **Validate determinism** by running the same query 10 times and comparing outputs:
   ```python
   outputs = [run_agent(query) for _ in range(10)]
   assert len(set(outputs)) == 1, f"Non-deterministic output detected: {set(outputs)}"
   ```

4. **Document determinism limits** in a comment:
   ```python
   # NOTE: temperature=0 + seed=42 provides near-deterministic output.
   # Minor floating-point variations across GPU parallel execution may occasionally
   # produce 1-2 token differences. For 100% reproducibility in tests, use MockModel.
   ```

5. **For evaluation runs**, always set the same seed to minimize methodological variance.

6. **For production classification agents** where determinism is required,
   additionally validate via `after_agent_callback` that the output is one of the
   expected values.

## Input Format

```python
{
  "model": "gemini-2.0-flash",
  "temperature": 0.0,
  "top_k": 1,
  "top_p": 1.0,
  "max_tokens": 256,
  "stop_sequences": [],
  "stream": False,
  "response_schema": {},
  "deterministic": True
}
```

## Output Format

```python
{
  "model_used": "gemini-2.0-flash",
  "configuration_applied": {
    "temperature": 0.0,
    "top_k": 1,
    "top_p": 1.0,
    "seed": 42
  },
  "output_valid": True,
  "structured_output": {},
  "determinism_validated": True
}
```

## Error Handling

- **Still getting variance at temperature=0**: Add `seed` parameter. Variance
  at temperature=0 comes from GPU floating-point non-determinism, not from
  sampling randomness. `seed` provides an additional reproducibility anchor.
- **`seed` not supported by model endpoint**: Some model variants may not
  support `seed`. Remove it and document the limitation. Use mocked responses
  for tests requiring 100% reproducibility.
- **Model behavior changes after model update** (even with same seed):
  Model weights change across versions. Lock the model version string and
  re-validate after any upgrade.
- **Deterministic config produces poor quality** (too rigid): Use temperature=0.1
  instead of 0.0 if pure greedy decoding produces repetitive or looping outputs.
- **Long outputs drift** even at temperature=0: Greedy decoding can loop on
  long generations. Add `max_output_tokens` and `stop_sequences` to prevent runaway.

## Examples

### Example 1 — Deterministic classification agent

```python
from google.adk.agents import LlmAgent
from google.genai.types import GenerateContentConfig
from pydantic import BaseModel
from typing import Literal

class IntentClassification(BaseModel):
    intent: Literal["refund", "order_status", "complaint", "product_inquiry", "other"]
    confidence: Literal["high", "medium", "low"]
    reasoning: str

DETERMINISTIC_CONFIG = GenerateContentConfig(
    temperature=0.0,
    top_k=1,
    top_p=1.0,
    seed=42,
    max_output_tokens=256,
)

classifier = LlmAgent(
    name="intent_classifier",
    model="gemini-2.0-flash",
    instruction="""
    Classify the customer's message into exactly one intent category.
    Provide a confidence level and brief reasoning.
    Be consistent — the same message must always produce the same classification.
    """,
    generate_content_config=DETERMINISTIC_CONFIG,
    output_schema=IntentClassification,
    output_key="intent_classification",
)
```

### Example 2 — Determinism validation test

```python
import asyncio
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai.types import Content, Part

async def validate_determinism(agent, queries: list[str], n_runs: int = 5) -> dict:
    """Validates that repeated runs of the same queries produce identical outputs."""
    results = {}
    session_service = InMemorySessionService()
    runner = Runner(agent=agent, app_name="test", session_service=session_service)

    for query in queries:
        outputs = set()
        for i in range(n_runs):
            session = await session_service.create_session(app_name="test", user_id=f"test_user_{i}")
            msg = Content(parts=[Part(text=query)])
            for event in runner.run(user_id=f"test_user_{i}", session_id=session.id, new_message=msg):
                if event.is_final_response() and event.content and event.content.parts:
                    outputs.add(event.content.parts[0].text or "")
        results[query] = {
            "is_deterministic": len(outputs) == 1,
            "unique_outputs": len(outputs),
        }
    return results
```

### Example 3 — Reproducible evaluation configuration

```python
from google.adk.evaluation.agent_evaluator import AgentEvaluator
from google.genai.types import GenerateContentConfig

# Apply deterministic config to agent for eval reproducibility
EVAL_CONFIG = GenerateContentConfig(temperature=0.0, seed=42)

# Use the same config in every eval run for comparable results
async def run_reproducible_eval(agent_module: str, eval_file: str):
    """Runs evaluation with deterministic config for reproducible measurement."""
    await AgentEvaluator.evaluate(
        agent_module=agent_module,
        eval_dataset_file_path_or_dir=eval_file,
    )
```

## Edge Cases

- **Greedy decoding loops** on repetitive content: Add stop sequences or set
  `max_output_tokens` to prevent infinite repetition at temperature=0.
- **Different outputs on different hardware** (CPU vs. GPU): Floating-point
  determinism is hardware-dependent. Standardize the deployment hardware for
  reproducible production behavior.
- **Seed causes degraded quality**: The seed affects sampling randomness. If
  `temperature=0`, the seed has minimal effect since greedy decoding is used.
  The seed matters more when temperature > 0.
- **Multi-turn sessions**: Each turn with `temperature=0` and the same pattern is
  deterministic given the same session history. Ensure session history is identical
  across repeated runs for full session-level determinism.

## Integration Notes

- **`google-adk-temperature-tuning`**: temperature=0 is the primary determinism
  control. This skill is a specialized application of temperature tuning.
- **`google-adk-top-k-sampling`**: top_k=1 reinforces temperature=0 greedy behavior.
- **`google-adk-automated-evaluation`**: Run evals with deterministic config to
  reduce metric variance across runs, making regression detection more reliable.
- **`google-adk-structured-json-output`**: Combine with structured output for
  maximum consistency in schema-constrained extraction tasks.
