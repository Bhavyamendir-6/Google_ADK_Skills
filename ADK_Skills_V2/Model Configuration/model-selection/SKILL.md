---
name: model-selection
description: >
  Use this skill when choosing which Gemini or Vertex AI model to configure
  for a Google ADK LlmAgent. Covers comparing model variants by capability,
  context window, cost, and latency; selecting the right model for the agent's
  task complexity; configuring the model field on LlmAgent; and switching
  models across environments or pipeline stages.
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

This skill governs model selection for Google ADK `LlmAgent`. The model choice
is the single highest-impact configuration decision — it determines the agent's
reasoning capability, context window, latency, and cost per invocation. This
skill provides a decision framework for selecting from Gemini model variants,
configuring the `model` field on `LlmAgent`, and handling model-specific
behavior differences. It prevents over-provisioning (using `gemini-pro` when
`gemini-flash` suffices) and under-provisioning (using a weak model for
complex multi-step reasoning).

## When to Use

- When initializing a new `LlmAgent` and choosing its `model` field.
- When the current model is too slow, too expensive, or too capable for the
  agent's actual task complexity.
- When a model upgrade or downgrade is being evaluated.
- When different sub-agents in a pipeline need different capability levels.
- When the agent requires multimodal input (images, audio, video) requiring a
  specific model variant.
- When a long context window is needed (e.g., > 32K tokens of session history
  or document context).

## When NOT to Use

- Do not select a model without first profiling the agent's actual token usage
  and latency requirements.
- Do not use `gemini-2.0-pro` by default — always start with `gemini-2.0-flash`
  and upgrade only when evaluation scores require it.
- Do not change the model in production without running the full evaluation suite
  on the new model first.

## Google ADK Context

- **`LlmAgent.model`**: A string identifying the Gemini model. Examples:
  - `"gemini-2.0-flash"` — Fast, cost-effective, 1M token context.
  - `"gemini-2.0-flash-lite"` — Lightest, fastest, lowest cost.
  - `"gemini-2.0-pro"` — Highest capability, slower, 2M token context.
  - `"gemini-1.5-flash"` — Stable, production-ready flash variant.
- **Vertex AI model strings**: In Vertex AI deployments, use full resource names:
  `"projects/{project}/locations/{location}/publishers/google/models/gemini-2.0-flash"`.
- **Model capability tiers**:
  - **Flash / Flash-Lite**: Simple tools, low-complexity reasoning, high throughput.
  - **Pro**: Complex multi-step reasoning, multi-tool orchestration, code generation.
- **`generate_content_config`**: Model-level inference config passed alongside
  the model — temperature, top_k, top_p, max_output_tokens.
- **Context window**: Flash = 1M tokens. Pro = 2M tokens. Plan session history
  management accordingly.
- **Multimodal**: All Gemini 2.0 variants support text, image, audio, video, and
  code inputs.

## Capabilities

- Selects the optimal Gemini model variant for agent task complexity.
- Reduces cost by matching model capability to task requirements.
- Reduces latency by using flash models for simple tasks.
- Configures model field on `LlmAgent` with correct model string.
- Applies different models to different sub-agents in a pipeline.
- Handles model-specific capability differences in instruction design.

## Step-by-Step Instructions

1. **Assess task complexity** of the agent:
   - Simple tool calls, retrieval, classification → `gemini-2.0-flash`
   - Complex multi-step reasoning, code generation, analysis → `gemini-2.0-pro`
   - Maximum throughput, minimum latency → `gemini-2.0-flash-lite`

2. **Assess context window needs**:
   - Session history < 100K tokens → Any model.
   - Session history up to 1M tokens → `gemini-2.0-flash` or `gemini-2.0-pro`.
   - Up to 2M tokens → `gemini-2.0-pro`.

3. **Check multimodal requirements**:
   - Text only → Any model.
   - Image, audio, video → All Gemini 2.0 variants support multimodal.

4. **Start with Flash** and validate with evaluation:
   ```python
   agent = LlmAgent(name="my_agent", model="gemini-2.0-flash", ...)
   ```

5. **Run evaluation** (`AgentEvaluator`) on Flash. If `tool_trajectory_avg_score`
   or `response_match_score` is below threshold, upgrade to Pro:
   ```python
   agent = LlmAgent(name="my_agent", model="gemini-2.0-pro", ...)
   ```

6. **In multi-agent pipelines**, apply the principle of least capability:
   ```python
   # Sub-agents doing simple retrieval
   data_agent = LlmAgent(model="gemini-2.0-flash", ...)
   # Root agent doing complex orchestration
   root_agent = LlmAgent(model="gemini-2.0-pro", ...)
   ```

7. **Store model version in `app:` state** for auditability:
   ```python
   app_state = {"app:model_version": "gemini-2.0-flash", "app:model_selection_date": "2024-03-15"}
   ```

8. **Validate with benchmarks** before deploying a model change. Record P50/P95
   latency and `response_match_score` before and after.

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
  "configuration_applied": {"model": "gemini-2.0-flash"},
  "output_valid": True,
  "structured_output": {}
}
```

## Error Handling

- **Invalid model string**: `LlmAgent.__init__` raises immediately. Use the
  exact strings from the Gemini API documentation.
- **Model not available in region**: Vertex AI models are region-specific. If
  the model raises a `NotFound` error, check the deployment region.
- **Model quota exceeded** (`ResourceExhausted`): Implement exponential backoff.
  Consider using `gemini-2.0-flash` as a fallback for `gemini-2.0-pro` quota
  exhaustion.
- **Context window exceeded**: ADK raises a token limit error. Reduce session
  history, apply context summarization, or switch to a larger context model.
- **Model changed behavior after API update**: Re-run the full evaluation suite
  after any Google model update. Store the commit hash of the evaluation run.

## Examples

### Example 1 — Model selection by task complexity

```python
from google.adk.agents import LlmAgent, SequentialAgent

# Flash for simple data retrieval
catalog_searcher = LlmAgent(
    name="catalog_searcher",
    model="gemini-2.0-flash",  # Fast, low cost for retrieval
    instruction="Search the product catalog and return matching items.",
    tools=[search_catalog],
)

# Pro for complex analysis and report generation
report_generator = LlmAgent(
    name="report_generator",
    model="gemini-2.0-pro",  # Best reasoning for complex synthesis
    instruction="Analyze the search results and generate a detailed investment report.",
)

pipeline = SequentialAgent(
    name="analysis_pipeline",
    sub_agents=[catalog_searcher, report_generator],
)
```

### Example 2 — Environment-based model selection

```python
import os
from google.adk.agents import LlmAgent

def get_model_for_env() -> str:
    """Selects model based on environment. Saves cost in development."""
    env = os.getenv("DEPLOYMENT_ENV", "dev")
    return {
        "prod": "gemini-2.0-pro",
        "staging": "gemini-2.0-flash",
        "dev": "gemini-2.0-flash-lite",
    }.get(env, "gemini-2.0-flash")

agent = LlmAgent(
    name="env_adaptive_agent",
    model=get_model_for_env(),
    instruction="You are a customer service assistant.",
    tools=[handle_inquiry],
)
```

### Example 3 — Model fallback on quota exhaustion

```python
import asyncio
from google.adk.runners import Runner
from google.genai.types import Content, Part
from google.api_core.exceptions import ResourceExhausted

async def run_with_fallback(primary_runner: Runner, fallback_runner: Runner,
                             user_id: str, session_id: str, query: str) -> str:
    """Runs agent with Pro model; falls back to Flash on quota exhaustion."""
    msg = Content(parts=[Part(text=query)])
    try:
        for event in primary_runner.run(user_id=user_id, session_id=session_id, new_message=msg):
            if event.is_final_response():
                return event.content.parts[0].text if event.content else ""
    except ResourceExhausted:
        import logging
        logging.warning("Pro model quota exhausted. Using Flash fallback.")
        for event in fallback_runner.run(user_id=user_id, session_id=session_id, new_message=msg):
            if event.is_final_response():
                return event.content.parts[0].text if event.content else ""
    return ""
```

## Edge Cases

- **Model upgraded by Google mid-deployment**: Pin model versions where possible.
  Monitor Google's model release notes for behavioral changes.
- **Flash model insufficient for complex task**: If `tool_trajectory_avg_score`
  is consistently < 0.8 on Flash, it's a signal to upgrade to Pro.
- **Very long inputs** (> 500K tokens): Only `gemini-2.0-flash` (1M) or
  `gemini-2.0-pro` (2M) can handle this. Choose based on reasoning needs.
- **Multi-modal input with unsupported format**: Check Gemini's supported formats
  before sending. Unsupported formats cause a `INVALID_ARGUMENT` error.

## Integration Notes

- **`google-adk-temperature-tuning`**: Temperature is a separate post-selection
  configuration. Apply after model is chosen.
- **`google-adk-token-limits`**: Token limit configuration is model-dependent.
  Apply limits after selecting the model.
- **`google-adk-performance-benchmarking`**: Benchmark both Flash and Pro on
  the actual task before committing to a model in production.
- **`google-adk-cost-tracking`**: Cost per token varies significantly by model.
  Track cost before and after a model switch.
