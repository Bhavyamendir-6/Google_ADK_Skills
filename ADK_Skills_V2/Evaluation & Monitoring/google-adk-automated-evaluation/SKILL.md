---
name: google-adk-automated-evaluation
description: >
  Use this skill to integrate Google ADK agent evaluation into automated
  CI/CD pipelines using AgentEvaluator, pytest, and the adk eval CLI.
  Covers creating eval datasets (.test.json, .evalset.json), running evaluations
  programmatically, interpreting results, and triggering regressions checks
  on every agent code change.
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

This skill implements fully automated evaluation of Google ADK agents as part
of a CI/CD pipeline. It covers creating structured evaluation datasets,
executing them with `AgentEvaluator.evaluate()` or `adk eval`, interpreting
scores against thresholds, and detecting regressions before they reach
production. It ensures agent quality is verified continuously — not just at
initial development.

## When to Use

- When merging changes to an ADK agent and needing automated quality assurance.
- When setting up a CI pipeline (GitHub Actions, Cloud Build, Jenkins) for an
  ADK agent project.
- When running regression tests after model upgrades or tool modifications.
- When generating automated evaluation reports for production deployments.

## When NOT to Use

- Do not run LLM-judged metrics (`final_response_match_v2`, `rubric_based_*`) on
  every commit — use them on scheduled runs or pre-release gates only.
- Do not use automated evaluation as the sole quality gate for high-stakes agents —
  combine with human-in-the-loop review (`google-adk-human-in-the-loop-evaluation`).
- Do not run automated eval against `InMemorySessionService` in CI — use a
  deterministic test environment with fixed seeds.

## Google ADK Context

- **`AgentEvaluator.evaluate()`**: The primary Python API for running evaluations.
  Accepts `agent_module`, `eval_dataset_file_path_or_dir`, and optional
  `config_file_path`.
- **`.test.json` files**: Single-session eval cases. Ideal for unit testing.
  Checked by ADK for `.test.json` suffix.
- **`.evalset.json` files**: Multi-session eval datasets. Ideal for integration
  tests. Requires Vertex Gen AI Evaluation Service for LLM-judged metrics.
- **`test_config.json`**: Specifies criteria thresholds. Placed in the eval
  directory. Applied to all tests in that directory.
- **`adk eval` CLI**: Runs evalset files from the command line. Supports
  `--config_file_path` and `--print_detailed_results`.
- **Agent Execution Lifecycle**: `AgentEvaluator` initializes a fresh session
  per eval case, runs the agent, captures trajectory and final response, then
  scores them against the eval case's ground truth.
- **Pytest integration**: `@pytest.mark.asyncio` with `AgentEvaluator.evaluate()`
  is the standard automated test integration pattern.

## Capabilities

- Runs ADK agent evaluations programmatically via `AgentEvaluator.evaluate()`.
- Executes eval runs via `adk eval` CLI in CI/CD shell scripts.
- Creates `.test.json` unit test files for fast, frequent evaluation.
- Creates `.evalset.json` integration test files for complex multi-turn sessions.
- Configures pass/fail thresholds via `test_config.json`.
- Detects regressions in tool trajectory and response quality.
- Generates detailed result reports with `--print_detailed_results`.
- Runs specific eval cases by name from evalset files.

## Step-by-Step Instructions

1. **Organize eval directory structure**:
   ```
   tests/
   └── eval/
       ├── test_config.json
       ├── unit/
       │   ├── device_control.test.json
       │   └── user_auth.test.json
       └── integration/
           └── booking_flow.evalset.json
   ```

2. **Create a `.test.json` unit eval case**:
   ```json
   {
     "eval_set_id": "device_control_unit",
     "eval_cases": [{
       "eval_id": "turn_off_device",
       "conversation": [{
         "invocation_id": "inv_001",
         "user_content": {"parts": [{"text": "Turn off device_2 in Bedroom"}], "role": "user"},
         "final_response": {"parts": [{"text": "I have set device_2 status to off."}], "role": "model"},
         "intermediate_data": {
           "tool_uses": [{"name": "set_device_info", "args": {"location": "Bedroom", "device_id": "device_2", "status": "OFF"}}],
           "intermediate_responses": []
         }
       }],
       "session_input": {"app_name": "home_agent", "user_id": "test_user", "state": {}}
     }]
   }
   ```

3. **Configure `test_config.json`**:
   ```json
   {"criteria": {"tool_trajectory_avg_score": 1.0, "response_match_score": 0.8}}
   ```

4. **Write pytest test cases**:
   ```python
   @pytest.mark.asyncio
   async def test_device_control():
       await AgentEvaluator.evaluate(
           agent_module="home_automation_agent",
           eval_dataset_file_path_or_dir="tests/eval/unit/device_control.test.json",
           config_file_path="tests/eval/test_config.json",
       )
   ```

5. **Add to CI pipeline** (GitHub Actions example):
   ```yaml
   - name: Run ADK Agent Evaluation
     run: |
       pip install google-adk pytest pytest-asyncio
       adk eval home_automation_agent/__init__.py \
         tests/eval/integration/booking_flow.evalset.json \
         --config_file_path=tests/eval/test_config.json \
         --print_detailed_results
   ```

6. **Interpret results**: A non-zero exit code from `adk eval` or a pytest
   `AssertionError` from `AgentEvaluator` indicates at least one eval case failed.

7. **Run subset of evals** from an evalset:
   ```bash
   adk eval agent/__init__.py evals.evalset.json:eval_1,eval_2
   ```

8. **Store results** in a CI artifact or log store for trend analysis.

## Input Format

```python
{
  "agent_id": "home_automation_agent",
  "session_id": "eval_auto_run_001",
  "tool_name": "set_device_info",
  "execution_time_ms": 312,
  "status": "success",
  "error": None,
  "metadata": {
    "eval_run_id": "ci_run_20240315_142300",
    "git_commit": "abc1234",
    "eval_file": "tests/eval/unit/device_control.test.json"
  }
}
```

## Output Format

```python
{
  "metric_recorded": True,
  "evaluation_result": "pass",
  "latency_ms": 312,
  "cost_estimate": 0.0,
  "issues_detected": [],
  "eval_run_id": "ci_run_20240315_142300",
  "passed_cases": 8,
  "failed_cases": 0,
  "total_cases": 8
}
```

## Error Handling

- **`agent_module` not found**: `ModuleNotFoundError`. Check that the agent
  module path is correct and `__init__.py` exports `root_agent`.
- **`.test.json` schema mismatch**: Run `AgentEvaluator.migrate_eval_data_to_new_schema()`
  to convert legacy eval files to the Pydantic-backed schema.
- **Vertex Gen AI Evaluation API unavailable**: Retry 3 times with exponential
  backoff. If unavailable, skip LLM-judged metrics and report `"warning"`.
- **Eval case timeout** (agent takes too long): Set a `pytest` timeout with
  `@pytest.mark.timeout(60)`. Log the session ID for debugging.
- **Partial results** (some eval cases crash): `AgentEvaluator` continues the
  remaining cases. Inspect the exception and mark crashed cases as `"fail"`.
- **Non-deterministic scores near threshold**: Add `±0.05` buffer to thresholds
  to avoid flaky CI failures from model output variance.

## Examples

### Example 1 — Full pytest test suite with multiple eval files

```python
import pytest
from google.adk.evaluation.agent_evaluator import AgentEvaluator

AGENT = "home_automation_agent"
CONFIG = "tests/eval/test_config.json"

@pytest.mark.asyncio
async def test_device_control_unit():
    await AgentEvaluator.evaluate(
        agent_module=AGENT,
        eval_dataset_file_path_or_dir="tests/eval/unit/device_control.test.json",
        config_file_path=CONFIG,
    )

@pytest.mark.asyncio
async def test_user_auth_unit():
    await AgentEvaluator.evaluate(
        agent_module=AGENT,
        eval_dataset_file_path_or_dir="tests/eval/unit/user_auth.test.json",
        config_file_path=CONFIG,
    )

@pytest.mark.asyncio
async def test_full_booking_integration():
    """Integration test — runs less frequently, uses evalset."""
    await AgentEvaluator.evaluate(
        agent_module=AGENT,
        eval_dataset_file_path_or_dir="tests/eval/integration/booking_flow.evalset.json",
        config_file_path="tests/eval/integration_config.json",
    )
```

### Example 2 — GitHub Actions CI workflow

```yaml
name: ADK Agent Evaluation CI

on: [push, pull_request]

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"
      - name: Install dependencies
        run: pip install google-adk pytest pytest-asyncio
      - name: Run unit evaluations
        run: pytest tests/eval/unit/ -v --tb=short
      - name: Run CLI evalset on schedule
        if: github.event_name == 'schedule'
        run: |
          adk eval home_automation_agent tests/eval/integration/booking.evalset.json \
            --config_file_path=tests/eval/integration_config.json \
            --print_detailed_results
```

### Example 3 — Migrating legacy eval files

```python
from google.adk.evaluation.agent_evaluator import AgentEvaluator

AgentEvaluator.migrate_eval_data_to_new_schema(
    current_test_data_file="tests/eval/old_format.test.json",
    output_file="tests/eval/migrated.test.json",
)
```

## Edge Cases

- **Zero eval cases in file**: `AgentEvaluator` completes with no results.
  Add a validation check: `assert len(eval_cases) > 0` before running.
- **Multi-turn evalset with state dependencies**: Ensure `session_input.state`
  is populated correctly for each eval case if the agent needs initial state.
- **Flaky tests near score threshold**: Use `response_match_score: 0.7` instead
  of `0.8` for highly variable responses. Document the reason in the config.
- **Eval run on a new agent** (no baseline): Run once to establish baseline
  scores, then set thresholds at `baseline - 0.1` for the CI gate.

## Integration Notes

- **`google-adk-agent-evaluation-metrics`**: Supplies the metric selection and
  `test_config.json` content that this skill executes.
- **`google-adk-human-in-the-loop-evaluation`**: Should be run after automated
  evaluation flags `"warning"` or borderline `"fail"` cases for human review.
- **ADK Web UI**: Use `adk web` to interactively build eval cases from real
  sessions, then export them as `.evalset.json` for automated runs.
- **Cloud Build / Vertex AI Pipelines**: `adk eval` CLI is the appropriate
  tool for integrating into GCP-native CI/CD pipelines.
