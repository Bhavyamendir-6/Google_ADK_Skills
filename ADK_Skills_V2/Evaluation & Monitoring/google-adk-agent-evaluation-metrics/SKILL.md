---
name: google-adk-agent-evaluation-metrics
description: >
  Use this skill when defining, selecting, and applying evaluation metrics for
  a Google ADK agent. Covers ADK's built-in criteria (tool_trajectory_avg_score,
  response_match_score, final_response_match_v2, rubric-based metrics,
  hallucinations_v1, safety_v1), configuring test_config.json, and choosing
  the right metric per evaluation goal.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: evaluation-and-monitoring
---

## Purpose

This skill defines the correct selection and configuration of evaluation metrics
for Google ADK agents. ADK agents are probabilistic systems — traditional
pass/fail unit tests are insufficient. This skill governs how to choose metrics
that assess both the agent's **trajectory** (the sequence of tool calls and
reasoning steps) and its **final response** (quality, correctness, safety),
ensuring the evaluation framework reflects real production requirements.

## When to Use

- When setting up a new agent evaluation pipeline and deciding which metrics to use.
- When configuring a `test_config.json` for `AgentEvaluator.evaluate()` or `adk eval`.
- When the current metrics do not reflect the agent's actual quality requirements.
- When adding hallucination detection or safety checks to an existing evaluation.
- When preparing an agent for CI/CD integration with automated score thresholds.

## When NOT to Use

- Do not apply LLM-based metrics (`final_response_match_v2`, `rubric_based_*`)
  in high-frequency CI pipelines — they are slow and costly. Use
  `tool_trajectory_avg_score` and `response_match_score` for CI speed.
- Do not use metrics that require reference responses with user-simulation evals —
  only `hallucinations_v1` and `safety_v1` support user-simulation mode.
- Do not set `tool_trajectory_avg_score` below 1.0 for safety-critical agents —
  the tool call sequence must be exact.

## Google ADK Context

- **ADK Evaluation Framework**: `google.adk.evaluation.agent_evaluator.AgentEvaluator`
  runs eval cases from `.test.json` or `.evalset.json` files using configurable
  criteria defined in `test_config.json`.
- **Built-in Criteria**:
  - `tool_trajectory_avg_score`: Exact match of tool call trajectory (name + args).
  - `response_match_score`: ROUGE-1 similarity of final response to reference.
  - `final_response_match_v2`: LLM-judged semantic equivalence to reference.
  - `rubric_based_final_response_quality_v1`: LLM-judged quality via custom rubrics.
  - `rubric_based_tool_use_quality_v1`: LLM-judged tool use ordering and correctness.
  - `hallucinations_v1`: Detects unsupported claims vs. available context.
  - `safety_v1`: Checks response for harmful content.
- **Default criteria** (when `test_config.json` is absent):
  - `tool_trajectory_avg_score`: 1.0 (exact match required)
  - `response_match_score`: 0.8 (80% ROUGE-1 similarity)
- **Agent Execution Lifecycle**: Metrics are applied post-invocation, comparing
  `intermediate_data.tool_uses` and `final_response` against expected values.
- **Vertex Gen AI Evaluation Service**: Required for LLM-judged metrics
  (`final_response_match_v2`, `rubric_based_*`, `hallucinations_v1`, `safety_v1`).

## Capabilities

- Selects the correct metric per evaluation goal (CI speed vs. quality depth).
- Configures custom thresholds in `test_config.json`.
- Combines multiple metrics in a single evaluation run.
- Detects trajectory deviations (wrong tool calls, wrong order, wrong args).
- Measures semantic response quality using LLM-as-judge.
- Evaluates response groundedness and safety automatically.
- Integrates metric configuration into ADK `pytest` and `adk eval` pipelines.

## Step-by-Step Instructions

1. **Define evaluation goals** for the agent:
   - Need CI/CD speed? → `tool_trajectory_avg_score` + `response_match_score`
   - Need semantic quality? → `final_response_match_v2`
   - No reference responses? → `rubric_based_final_response_quality_v1`
   - Check tool ordering? → `rubric_based_tool_use_quality_v1`
   - Detect hallucinations? → `hallucinations_v1`
   - Enforce safety? → `safety_v1`

2. **Create `test_config.json`** in the eval directory:
   ```json
   {
     "criteria": {
       "tool_trajectory_avg_score": 1.0,
       "response_match_score": 0.8
     }
   }
   ```

3. **Create eval cases** in `.test.json` with `intermediate_data.tool_uses`
   and `final_response` populated as ground-truth reference data.

4. **Run evaluation** using `AgentEvaluator.evaluate()` in pytest:
   ```python
   await AgentEvaluator.evaluate(
       agent_module="my_agent",
       eval_dataset_file_path_or_dir="tests/eval/my_agent.test.json",
       config_file_path="tests/eval/test_config.json",
   )
   ```

5. **Analyze results**: Check scores against thresholds. Any criterion below
   its threshold causes the eval case to fail.

6. **Iterate on metrics**: If `tool_trajectory_avg_score` fails due to benign
   variation in tool args, lower the threshold or switch to
   `rubric_based_tool_use_quality_v1` with a rubric that checks for the
   essential constraints only.

7. **Record threshold decisions** in a `METRICS.md` in the eval directory
   explaining why each threshold was chosen.

## Input Format

```python
{
  "agent_id": "home_automation_agent",
  "session_id": "eval_session_001",
  "tool_name": "set_device_info",
  "execution_time_ms": 452,
  "status": "success",
  "error": None,
  "metadata": {
    "eval_criteria": {
      "tool_trajectory_avg_score": 1.0,
      "response_match_score": 0.8
    }
  }
}
```

## Output Format

```python
{
  "metric_recorded": True,
  "evaluation_result": "pass",
  "scores": {
    "tool_trajectory_avg_score": 1.0,
    "response_match_score": 0.87
  },
  "latency_ms": 452,
  "cost_estimate": 0.0,
  "issues_detected": []
}
```

On failure:
```python
{
  "metric_recorded": True,
  "evaluation_result": "fail",
  "scores": {
    "tool_trajectory_avg_score": 0.5,
    "response_match_score": 0.91
  },
  "issues_detected": [
    "tool_trajectory_avg_score below threshold 1.0 (actual: 0.5): agent called wrong tool."
  ]
}
```

## Error Handling

- **Missing `test_config.json`**: ADK uses defaults (`tool_trajectory_avg_score: 1.0`,
  `response_match_score: 0.8`). Log a warning and document the defaults.
- **LLM-judged metric fails** (Vertex API unavailable): Catch `google.api_core.exceptions`
  and retry up to 3 times with backoff. Report eval as `"warning"` if retries
  exhaust — do not mark as `"fail"` due to infrastructure error.
- **Empty `tool_uses` in eval case**: `tool_trajectory_avg_score` defaults to 1.0
  (trivially satisfied). Validate that eval cases with tools expected have non-empty
  `intermediate_data.tool_uses`.
- **Invalid metric key in `test_config.json`**: ADK raises `ValueError`. Validate
  config keys against the supported criteria list before running.

## Examples

### Example 1 — pytest with custom metrics

```python
import pytest
from google.adk.evaluation.agent_evaluator import AgentEvaluator

@pytest.mark.asyncio
async def test_home_automation_trajectory():
    """Verifies exact tool trajectory for device control commands."""
    await AgentEvaluator.evaluate(
        agent_module="home_automation_agent",
        eval_dataset_file_path_or_dir="tests/eval/home_automation.test.json",
        config_file_path="tests/eval/test_config.json",
    )

@pytest.mark.asyncio
async def test_home_automation_safety():
    """Verifies agent responses are safe and not harmful."""
    await AgentEvaluator.evaluate(
        agent_module="home_automation_agent",
        eval_dataset_file_path_or_dir="tests/eval/safety_cases.evalset.json",
        config_file_path="tests/eval/safety_config.json",
    )
```

### Example 2 — test_config.json for CI pipeline

```json
{
  "criteria": {
    "tool_trajectory_avg_score": 1.0,
    "response_match_score": 0.8
  }
}
```

### Example 3 — test_config.json for deep quality evaluation

```json
{
  "criteria": {
    "final_response_match_v2": 0.7,
    "rubric_based_final_response_quality_v1": 0.8,
    "hallucinations_v1": 0.9,
    "safety_v1": 1.0
  }
}
```

### Example 4 — CLI evaluation

```bash
adk eval \
  my_agent/__init__.py \
  tests/eval/my_agent.evalset.json \
  --config_file_path=tests/eval/test_config.json \
  --print_detailed_results
```

## Edge Cases

- **Partial tool trajectory** (agent makes extra tool calls not in expected):
  `tool_trajectory_avg_score` measures exact match — extra calls reduce scores.
  Use `rubric_based_tool_use_quality_v1` with a rubric like "Tool A must be called"
  for flexible matching.
- **Non-deterministic final responses**: Set `response_match_score` threshold to
  0.6–0.7 to allow phrasing variation. Use `final_response_match_v2` for semantic
  equivalence checking.
- **User simulation evals**: Only `hallucinations_v1` and `safety_v1` are supported.
  Do NOT include trajectory or response match metrics in user simulation configs.
- **Missing reference responses**: Use rubric-based metrics — no ground truth needed.

## Integration Notes

- **`google-adk-automated-evaluation`**: This skill defines the metrics;
  `automated-evaluation` runs them in CI/CD pipelines.
- **Vertex Gen AI Evaluation Service**: Required for LLM-judged metrics.
  Set `GOOGLE_CLOUD_PROJECT` and authenticate with ADC before running.
- **ADK Web UI**: Supports interactive metric configuration via sliders for
  `tool_trajectory_avg_score` and `response_match_score`. Use for exploratory
  evaluation before finalizing `test_config.json`.
- **`google-adk-performance-benchmarking`**: Combines execution latency data
  with evaluation metrics to produce comprehensive agent quality reports.
