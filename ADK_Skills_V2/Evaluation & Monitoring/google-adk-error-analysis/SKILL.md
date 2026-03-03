---
name: google-adk-error-analysis
description: >
  Use this skill when a Google ADK agent produces errors, unexpected results,
  or evaluation failures that need systematic root-cause analysis. Covers
  classifying error types (tool errors, LLM refusals, trajectory deviations,
  timeout/rate-limit errors), correlating errors with session events, analyzing
  patterns across eval failures, and feeding insights back into the agent
  improvement workflow.
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

This skill performs systematic error analysis for Google ADK agents. When an
agent fails — whether in production (unexpected responses, tool failures) or
in evaluation (trajectory deviations, low scores) — this skill classifies the
error, traces it to its root cause using `session.events` and log data, identifies
patterns across multiple failure cases, and prescribes targeted fixes. It
closes the feedback loop between agent monitoring and agent improvement.

## When to Use

- When an ADK agent's evaluation score drops below threshold and the cause is
  unknown.
- When a tool returns an error dict and the agent cannot recover gracefully.
- When a production session produces an unexpected or incorrect final response.
- When automated evaluation detects a trajectory deviation (wrong tool called,
  wrong arguments, wrong order).
- When analyzing error patterns across multiple sessions to identify systemic
  issues.
- When the ADK Web UI Trace View shows unexpected model requests or responses.

## When NOT to Use

- Do not apply this skill to first-pass prototyping — wait until the agent has
  a baseline evaluation suite and production traffic.
- Do not analyze individual session errors in isolation without checking for
  cross-session patterns.
- Do not conflate evaluation metric failures with production errors —
  evaluation failures may be due to misaligned ground truth, not agent bugs.

## Google ADK Context

- **`session.events`**: The primary source of error analysis data. Each event
  contains the full sequence of model calls, tool calls, tool responses, and
  state changes within a session.
- **Error event types** in ADK:
  - **Tool error dict**: Tool returns `{"error": "...", "error_type": "..."}`.
  - **LLM refusal**: Model returns a content block with no function call when
    one was expected, or generates an off-topic response.
  - **Trajectory deviation**: `tool_trajectory_avg_score < 1.0` in eval.
  - **Session timeout**: Long tool calls or LLM calls time out.
  - **Rate-limit error**: `ResourceExhausted` from Gemini API.
  - **State corruption**: State key has wrong type or missing value.
- **ADK Web UI Trace View**: Inspect individual event request/response pairs
  to identify the exact step where the error occurred.
- **`event.actions.state_delta`**: Check state changes at the error point to
  detect state issues.
- **`google.adk.evaluation.agent_evaluator.AgentEvaluator`**: Provides
  structured failure output identifying which eval cases failed and why.
- **`adk eval --print_detailed_results`**: Prints per-case scores and actual
  vs. expected trajectory comparisons in the CLI.

## Capabilities

- Classifies ADK errors into types: tool error, LLM refusal, trajectory
  deviation, timeout, rate limit, state corruption.
- Traces errors to their root cause using `session.events`.
- Aggregates error counts by type, tool, and agent across multiple sessions.
- Correlates evaluation failures with specific eval case IDs and turns.
- Identifies systemic patterns (e.g., one tool consistently fails, one query
  type consistently causes trajectory deviation).
- Generates structured error reports with fix prescriptions.
- Feeds confirmed error patterns into updated eval datasets for regression testing.

## Step-by-Step Instructions

1. **Identify the error source**:
   - Production: Inspect `session.events` for the failing session.
   - Evaluation: Run `adk eval --print_detailed_results` or inspect
     `AgentEvaluator` output.

2. **Classify the error type**:
   ```python
   def classify_error(event) -> str:
       if event.content and event.content.parts:
           for part in event.content.parts:
               if hasattr(part, "function_response") and part.function_response:
                   response = part.function_response.response
                   if isinstance(response, dict) and "error" in response:
                       return f"tool_error:{response.get('error_type', 'unknown')}"
       if event.is_final_response() and event.author == "model":
           text = " ".join(p.text for p in event.content.parts if hasattr(p, "text") and p.text)
           if not text or "I cannot" in text or "I'm unable" in text:
               return "llm_refusal"
       return "unknown"
   ```

3. **Trace the error in `session.events`**:
   ```python
   for i, event in enumerate(session.events):
       error_type = classify_error(event)
       if error_type != "unknown":
           print(f"Error at event {i}: {error_type}")
           print(f"  Invocation: {event.invocation_id}")
           print(f"  Author: {event.author}")
           print(f"  State at error: {event.actions.state_delta}")
   ```

4. **Aggregate errors across sessions** by loading multiple session objects:
   ```python
   error_counts = {}
   for session in sessions:
       for event in session.events:
           et = classify_error(event)
           if et != "unknown":
               error_counts[et] = error_counts.get(et, 0) + 1
   ```

5. **Identify the pattern** from error_counts. E.g., if `tool_error:timeout`
   accounts for 80% of errors, the root cause is a slow external API.

6. **Prescribe and implement the fix**:
   - `tool_error:timeout` → Add retry logic + increase timeout.
   - `tool_error:invalid_input` → Improve input validation in the tool.
   - `llm_refusal` → Revise agent instruction to be clearer.
   - `trajectory_deviation` → Add more eval cases for this pattern and refine
     the agent's instruction or tool list.
   - `state_corruption` → Add state type validation in the affected tool.

7. **Add a regression eval case** for each confirmed error pattern:
   ```python
   # Add to eval dataset: a case that exercises the previously failing scenario
   eval_case = create_eval_case_from_session(failing_session_id)
   ```

8. **Verify the fix** by re-running the eval suite and confirming the error
   no longer appears.

## Input Format

```python
{
  "agent_id": "my_agent",
  "session_id": "sess_failing_123",
  "tool_name": "search_products",
  "execution_time_ms": 5200,
  "status": "failure",
  "error": "TimeoutError: search API did not respond within 5s",
  "metadata": {
    "error_type": "timeout",
    "eval_case_id": "eval_product_search_03",
    "trajectory_score": 0.5,
    "expected_tools": ["search_products", "format_results"],
    "actual_tools": ["search_products"]
  }
}
```

## Output Format

```python
{
  "metric_recorded": True,
  "evaluation_result": "fail",
  "latency_ms": 5200,
  "cost_estimate": 0.0,
  "issues_detected": [
    "tool_error:timeout in search_products (5200ms > 5000ms limit)",
    "trajectory_deviation: format_results not called after search_products"
  ],
  "error_analysis": {
    "error_type": "tool_error:timeout",
    "root_cause": "External search API response time exceeded 5s timeout",
    "affected_tool": "search_products",
    "pattern": "Occurs on queries with > 3 filter parameters",
    "fix_prescription": "Increase timeout to 10s; add retry for timeout errors; cache frequent queries",
    "regression_eval_added": True
  }
}
```

## Error Handling

- **Missing `session.events`** (session not found or in-memory backend):
  Fall back to log analysis. Search structured logs by `session_id` for tool
  error events.
- **Error classification returns `unknown`**: Inspect the raw event content
  manually via the ADK Web UI Trace View. Add a new classification rule for
  the discovered pattern.
- **Eval failure is misaligned ground truth** (not agent bug): Update the
  eval case's `final_response` or `tool_uses` to reflect correct expected
  behavior. Document the correction in the eval case metadata.
- **Error pattern spans multiple agents** (pipeline failure): Trace by
  `invocation_id` across all sub-agents' logs to find the originating agent.
- **Rate-limit errors in evaluation**: Implement exponential backoff in the
  benchmark/eval runner. Do not classify API rate-limit errors as agent bugs.
- **Storage failure analyzing large sessions**: Process events in batches of 100.

## Examples

### Example 1 — Error classification and analysis across sessions

```python
import json
from collections import Counter
from google.adk.sessions import DatabaseSessionService

async def analyze_errors(session_service, app_name: str, user_id: str, session_ids: list[str]) -> dict:
    """Analyzes error patterns across multiple ADK sessions."""
    error_counts = Counter()
    error_details = []

    for session_id in session_ids:
        session = await session_service.get_session(
            app_name=app_name, user_id=user_id, session_id=session_id
        )
        if not session:
            continue

        for event in session.events:
            error_type = classify_event_error(event)
            if error_type:
                error_counts[error_type] += 1
                error_details.append({
                    "session_id": session_id,
                    "invocation_id": event.invocation_id,
                    "error_type": error_type,
                    "author": event.author,
                    "timestamp": event.timestamp,
                })

    top_errors = error_counts.most_common(5)
    return {
        "total_sessions_analyzed": len(session_ids),
        "total_errors": sum(error_counts.values()),
        "error_distribution": dict(top_errors),
        "dominant_error": top_errors[0][0] if top_errors else None,
        "error_details": error_details[:20],  # Limit to 20 examples
    }


def classify_event_error(event) -> str | None:
    """Classifies an ADK event as an error if applicable."""
    if not event.content or not event.content.parts:
        return None

    for part in event.content.parts:
        # Tool error dict
        if hasattr(part, "function_response") and part.function_response:
            response = part.function_response.response
            if isinstance(response, dict) and "error" in response:
                return f"tool_error:{response.get('error_type', 'unexpected')}"

    # LLM refusal detection
    if event.is_final_response() and event.author == "model":
        text_parts = [p.text for p in event.content.parts if hasattr(p, "text") and p.text]
        if not text_parts:
            return "llm_no_response"
        full_text = " ".join(text_parts).lower()
        if any(phrase in full_text for phrase in ["i cannot", "i'm unable", "i can't", "not able to"]):
            return "llm_refusal"

    return None
```

### Example 2 — Trajectory deviation analysis from eval results

```python
import json

def analyze_trajectory_deviations(eval_results: list[dict]) -> dict:
    """Identifies patterns in trajectory deviations from eval results."""
    deviations = []

    for result in eval_results:
        if result.get("tool_trajectory_avg_score", 1.0) < 1.0:
            expected = result.get("expected_tools", [])
            actual = result.get("actual_tools", [])
            missing = set(expected) - set(actual)
            extra = set(actual) - set(expected)
            deviations.append({
                "eval_id": result["eval_id"],
                "score": result["tool_trajectory_avg_score"],
                "missing_tools": list(missing),
                "extra_tools": list(extra),
                "fix_hint": (
                    f"Agent skipped: {missing}" if missing else
                    f"Agent called unexpected: {extra}"
                )
            })

    # Count most missing tools
    from collections import Counter
    all_missing = [t for d in deviations for t in d["missing_tools"]]
    most_skipped = Counter(all_missing).most_common(5)

    return {
        "total_deviations": len(deviations),
        "most_frequently_skipped_tools": most_skipped,
        "deviation_details": deviations,
        "prescription": (
            f"Update agent instruction to explicitly require calling: {[t for t, _ in most_skipped[:3]]}"
            if most_skipped else "No dominant trajectory deviation found."
        )
    }
```

### Example 3 — Adding regression eval case from a failing session

```python
import json
from google.adk.sessions import DatabaseSessionService

async def create_regression_eval_case(session_service, app_name, user_id, session_id, output_path):
    """Creates a .test.json eval case from a failing production session for regression testing."""
    session = await session_service.get_session(
        app_name=app_name, user_id=user_id, session_id=session_id
    )

    if not session:
        raise ValueError(f"Session {session_id} not found")

    conversation = []
    for event in session.events:
        if event.author == "user" and event.content and event.content.parts:
            user_text = event.content.parts[0].text if event.content.parts else ""
            conversation.append({
                "invocation_id": event.invocation_id,
                "user_content": {"parts": [{"text": user_text}], "role": "user"},
                "final_response": None,
                "intermediate_data": {"tool_uses": [], "intermediate_responses": []},
            })
        elif event.is_final_response() and event.author == "model":
            if conversation and conversation[-1]["final_response"] is None:
                text = ""
                for part in (event.content.parts or []):
                    if hasattr(part, "text") and part.text:
                        text = part.text
                        break
                conversation[-1]["final_response"] = {"parts": [{"text": text}], "role": "model"}

    eval_case = {
        "eval_set_id": f"regression_{session_id}",
        "eval_cases": [{
            "eval_id": f"regression_{session_id}",
            "conversation": [c for c in conversation if c["final_response"]],
            "session_input": {"app_name": app_name, "user_id": user_id, "state": {}},
        }]
    }

    with open(output_path, "w") as f:
        json.dump(eval_case, f, indent=2, default=str)

    return output_path
```

## Edge Cases

- **Error appears only under load** (race condition): Cannot reproduce in single-run
  analysis. Enable parallel benchmarking and check for concurrent state writes.
- **Error is non-deterministic** (appears 10% of the time from model variance):
  Run the same query 20 times and measure error rate. If > 5%, it's a systematic
  issue requiring instruction or tool fix.
- **Eval failure is overly strict ground truth** (agent gave a correct but
  differently worded response): Use `final_response_match_v2` instead of
  `response_match_score`, or update the `final_response` in the eval case.
- **Error cascades across agents** (sub-agent error propagates to root agent):
  Trace by `invocation_id` to the originating sub-agent. Fix the sub-agent's
  error handling first.
- **Incomplete events** (session cut off by restart): Partial sessions are
  still analyzable up to the last appended event. Flag as incomplete in the
  error report.

## Integration Notes

- **`google-adk-automated-evaluation`**: Error analysis feeds confirmed bug
  cases back as new eval cases for regression testing.
- **`google-adk-logging`**: Structured tool error logs are the first data
  source for error classification when `session.events` is unavailable.
- **ADK Web UI Trace View**: Use for interactive per-event debugging. The Event,
  Request, Response, and Graph tabs reveal the exact point of failure.
- **`google-adk-human-in-the-loop-evaluation`**: Ambiguous errors (is this an
  agent bug or misaligned ground truth?) go to human reviewers.
- **`google-adk-observability`**: Error rates are tracked as OTel metrics.
  Set up alerts when `adk.tool.errors` counter exceeds a threshold.
