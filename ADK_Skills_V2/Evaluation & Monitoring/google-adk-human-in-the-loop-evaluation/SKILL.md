---
name: google-adk-human-in-the-loop-evaluation
description: >
  Use this skill when a Google ADK agent's output requires human review before
  being accepted as ground truth, approved for production, or used to update
  evaluation datasets. Covers capturing agent sessions via the ADK Web UI,
  human annotation of eval cases, escalation from automated evaluation, and
  approval gates for agent deployments.
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

This skill implements human-in-the-loop (HITL) evaluation for Google ADK agents.
It governs when and how human reviewers annotate agent sessions, override automated
evaluation results, create ground-truth eval cases from real conversations, and
approve agents for production deployment. HITL evaluation bridges the gap between
automated metric scores and actual user-perceived quality.

## When to Use

- When automated evaluation scores are borderline (near threshold) and a human
  must decide pass/fail.
- When creating ground-truth eval cases from real production sessions via the
  ADK Web UI.
- When an agent is being promoted to production and requires explicit human sign-off.
- When evaluating qualitative aspects (tone, helpfulness, trustworthiness) that
  automated metrics cannot capture.
- When a safety evaluation (`safety_v1`) flags a potential issue requiring human review.

## When NOT to Use

- Do not require human review on every evaluation run — limit to edge cases,
  borderline results, and new agent releases.
- Do not manually annotate eval cases for things automated metrics can reliably
  measure (e.g., exact tool trajectory matching).
- Do not use this skill as a replacement for automated evaluation — it
  supplements it, not replaces it.

## Google ADK Context

- **ADK Web UI (`adk web`)**: The primary interface for human evaluation.
  The **Eval tab** allows reviewers to add sessions as eval cases, edit expected
  responses, and run evaluations with configurable metrics.
- **Trace View in ADK Web UI**: Provides step-by-step inspection of agent execution
  — event data, model requests/responses, tool call graphs. Used by human reviewers
  to audit agent reasoning.
- **Eval Set management via UI**: Reviewers can create/select eval sets, add
  current sessions as eval cases, modify expected responses, and delete cases.
- **`session.events`**: The source of truth for human review — reviewers inspect
  the full event history to assess agent reasoning quality.
- **Agent Execution Lifecycle**: HITL review is applied post-execution, using
  the captured session events and agent output as review material.
- **`adk web <agent_path>`**: Launches the web UI for a specific agent for
  interactive human evaluation.

## Capabilities

- Captures production/staging agent sessions for human review via ADK Web UI.
- Enables human annotation of expected tool trajectories and final responses.
- Supports side-by-side comparison of actual vs. expected output in the UI.
- Implements approval gates: blocks deployment until a reviewer approves.
- Captures borderline automated eval failures for human adjudication.
- Creates high-quality eval cases from annotated real sessions.
- Supports escalation from automated pipelines to human review queues.

## Step-by-Step Instructions

1. **Launch ADK Web UI** for the target agent:
   ```bash
   adk web path/to/my_agent
   ```

2. **Interact with the agent** to create a representative session. Test
   scenarios that are critical or hard to cover automatically.

3. **Open the Eval tab** in the web UI and select or create an eval set.

4. **Click "Add current session"** to save the conversation as a new eval case.

5. **Review the captured eval case**:
   - Click the eval case ID to inspect it.
   - Click the **Edit** (pencil) icon to modify expected responses.
   - Add correct tool trajectories if the agent's actual trajectory was wrong.
   - Delete agent messages that shouldn't be part of the expected output.

6. **Use the Trace View** to inspect the agent's reasoning:
   - Click each trace row to see Event data, model Request, model Response, and
     the tool call Graph.
   - Identify any unexpected reasoning steps or hallucinations.

7. **Run evaluation on the annotated case** via the UI:
   - Select the eval case.
   - Click **Run Evaluation**.
   - Configure metric thresholds via the sliders.
   - Click **Start** and review the pass/fail result.

8. **For approval gates**: Implement a state machine:
   - Automated eval runs first.
   - If score < threshold → escalate to human reviewer queue.
   - Human reviews and marks `approved: True` or `requires_fix: True`.
   - Deployment proceeds only when `approved: True`.

9. **Export annotated eval cases** to `.test.json` for regression testing
   in CI pipelines.

## Input Format

```python
{
  "agent_id": "my_agent",
  "session_id": "sess_review_001",
  "tool_name": "search_products",
  "execution_time_ms": 780,
  "status": "success",
  "error": None,
  "metadata": {
    "review_requested_by": "automated_eval_pipeline",
    "automated_score": 0.62,
    "threshold": 0.8,
    "reason": "response_match_score below threshold",
    "reviewer": "alice@example.com"
  }
}
```

## Output Format

```python
{
  "metric_recorded": True,
  "evaluation_result": "pass",
  "latency_ms": 780,
  "cost_estimate": 0.0,
  "issues_detected": [],
  "human_review": {
    "reviewer": "alice@example.com",
    "decision": "approved",
    "notes": "Response is semantically correct despite low ROUGE-1 score.",
    "reviewed_at": "2024-03-15T10:30:00Z"
  }
}
```

## Error Handling

- **ADK Web UI not accessible**: Ensure `adk web` is running and the port
  is reachable. Check for firewall rules blocking the default port (8000).
- **Session not captured correctly**: Verify the session service is persistent
  (`DatabaseSessionService`) — `InMemorySessionService` sessions may not persist
  long enough for review.
- **Reviewer queue backlog**: Set SLA timers on review tasks. Auto-escalate
  to a secondary reviewer after 24h with no action.
- **Missing telemetry data** for review: If `session.events` is empty or
  incomplete, re-run the scenario manually before scheduling human review.
- **Conflicting reviewer decisions**: Implement a tie-breaking rule (e.g.,
  two reviewers must agree for approval, or senior reviewer has final say).

## Examples

### Example 1 — Automated eval escalation to human review

```python
import asyncio
from google.adk.evaluation.agent_evaluator import AgentEvaluator

THRESHOLD = 0.8

async def run_with_escalation(agent_module: str, eval_file: str, config_file: str):
    """Runs automated eval; escalates to HITL if score is borderline."""
    try:
        await AgentEvaluator.evaluate(
            agent_module=agent_module,
            eval_dataset_file_path_or_dir=eval_file,
            config_file_path=config_file,
        )
        print("Automated evaluation PASSED. No human review needed.")
    except AssertionError as e:
        score = parse_score_from_error(str(e))
        if THRESHOLD - 0.1 <= score < THRESHOLD:
            # Borderline — escalate to human review
            create_human_review_task(
                session_id=extract_session_id(str(e)),
                automated_score=score,
                reviewer="reviewer@example.com",
                reason="Borderline automated score — requires human adjudication",
            )
            print(f"Score {score} near threshold. Escalated to human review.")
        else:
            # Clearly failed — do not block on human review
            raise

def parse_score_from_error(error_msg: str) -> float:
    """Extracts numeric score from AgentEvaluator assertion error message."""
    import re
    match = re.search(r"(\d+\.\d+)", error_msg)
    return float(match.group(1)) if match else 0.0

def create_human_review_task(session_id, automated_score, reviewer, reason):
    """Logs a human review task to a ticketing system or queue."""
    import logging
    logging.getLogger("hitl").info({
        "session_id": session_id,
        "automated_score": automated_score,
        "reviewer": reviewer,
        "reason": reason,
    })
```

### Example 2 — Deployment approval gate

```python
from dataclasses import dataclass
from typing import Literal
import time

@dataclass
class ReviewDecision:
    reviewer: str
    decision: Literal["approved", "requires_fix", "pending"]
    notes: str
    reviewed_at: float = None

    def __post_init__(self):
        self.reviewed_at = self.reviewed_at or time.time()

def check_deployment_gate(review: ReviewDecision) -> bool:
    """Returns True if agent is approved for deployment."""
    if review.decision == "approved":
        print(f"Deployment approved by {review.reviewer} at {review.reviewed_at}")
        return True
    elif review.decision == "requires_fix":
        raise RuntimeError(f"Deployment blocked: {review.notes}")
    else:
        raise TimeoutError("Review is still pending. Deployment blocked.")
```

### Example 3 — Exporting annotated session as eval case

```python
import json
from google.adk.sessions import DatabaseSessionService

async def export_session_as_eval_case(session_service, app_name, user_id, session_id, output_path):
    """Exports a session as a .test.json eval case for CI regression testing."""
    session = await session_service.get_session(
        app_name=app_name, user_id=user_id, session_id=session_id
    )
    eval_case = {
        "eval_set_id": f"hitl_approved_{session_id}",
        "eval_cases": [{
            "eval_id": session_id,
            "conversation": [
                {
                    "invocation_id": e.invocation_id,
                    "user_content": e.content.__dict__ if e.author == "user" else None,
                    "final_response": e.content.__dict__ if e.is_final_response() else None,
                }
                for e in session.events
                if e.author in ("user", "agent")
            ],
            "session_input": {
                "app_name": session.app_name,
                "user_id": session.user_id,
                "state": {},
            }
        }]
    }
    with open(output_path, "w") as f:
        json.dump(eval_case, f, indent=2, default=str)
    return output_path
```

## Edge Cases

- **Reviewer approves a session with a known bug**: Implement a second-reviewer
  requirement for agents with safety or compliance implications.
- **Session contains PII**: Scrub PII from eval cases before exporting for
  regression testing.
- **Long sessions with 50+ turns**: Use the Trace View's invocation grouping to
  navigate efficiently. Focus review on turns where scores were lowest.
- **Incomplete session** (agent crashed mid-conversation): Tag as `incomplete`
  in the review system. Do not use as a ground-truth eval case.

## Integration Notes

- **ADK Web UI (Eval Tab)**: Primary tool for HITL evaluation. Launch with
  `adk web <agent_path>`. The Trace View provides full execution audit capability.
- **`google-adk-automated-evaluation`**: Runs first; escalates borderline or
  failed cases to this skill's human review process.
- **`google-adk-logging`**: Provides the session event and trace data that human
  reviewers inspect during the review process.
- **`google-adk-error-analysis`**: Feeds systematic error patterns discovered
  during HITL review back into the automated evaluation dataset.
