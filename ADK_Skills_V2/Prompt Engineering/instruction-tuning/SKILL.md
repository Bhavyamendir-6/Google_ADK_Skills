---
name: instruction-tuning
description: >
  Use this skill when iteratively refining a Google ADK LlmAgent's instruction
  field to improve task performance, tool usage accuracy, reasoning quality, and
  output consistency. Covers systematic instruction revision based on evaluation
  results, tool trajectory analysis, and behavioral testing to close the gap
  between actual and desired agent behavior.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: prompt-engineering
---

## Purpose

This skill implements a systematic instruction tuning workflow for Google ADK
`LlmAgent`. When an agent's behavior deviates from the expected trajectory,
produces incorrect outputs, misuses tools, or fails evaluation metrics,
instruction tuning provides a disciplined approach to diagnosing the issue in
the `instruction` string and revising it iteratively until the agent meets its
behavioral requirements. It treats the `instruction` as a precision instrument
that must be calibrated against real evaluation data.

## When to Use

- When `tool_trajectory_avg_score` in automated evaluation falls below threshold,
  indicating the agent is calling wrong tools or in the wrong order.
- When `response_match_score` is low, suggesting the agent's output doesn't match
  the expected format or content.
- When the agent ignores a tool it should be using (not in trajectory).
- When the agent produces verbose responses when brevity is required.
- When transitioning an agent from prototype to production and needing to harden
  its behavioral contract.

## When NOT to Use

- Do not tune instructions to compensate for tool bugs — fix the tool first.
- Do not tune instructions without a baseline evaluation dataset to measure
  improvement — you need ground truth to know if changes help.
- Do not make more than 2–3 changes per iteration — isolate variables to
  understand what each change does.

## Google ADK Context

- **`LlmAgent.instruction`**: The instruction string is the primary tuning lever.
  Changes here directly control the model's behavior without changing tools or model.
- **ADK Evaluation Framework** (`AgentEvaluator`): The feedback loop for
  instruction tuning. Run evals before and after each instruction change.
- **`tool_trajectory_avg_score`**: Measures whether the agent called the right
  tools in the right order. Low score → instruction needs explicit tool call guidance.
- **`response_match_score`**: Measures output quality. Low score → instruction
  needs output format, length, or content constraints.
- **Tool docstrings**: The LLM uses tool docstrings to decide when to call a
  tool. If the agent misuses a tool, improving its docstring can be as effective
  as tuning the instruction.
- **ADK Web UI Trace View**: Show exact model requests sent to the LLM — use
  these to understand what the LLM "sees" when it makes the wrong decision.
- **`InstructionProvider`**: Allows conditional instruction logic — different
  instruction content for different session states.

## Capabilities

- Diagnoses instruction weaknesses from evaluation metric failures.
- Applies targeted instruction revisions (adding explicit tool call sequences,
  tightening output constraints, adding MUST/MUST NOT directives).
- Implements iterative A/B testing of instruction variants against eval datasets.
- Identifies when tool docstrings (not the instruction) are the root cause.
- Uses `InstructionProvider` to apply context-specific instruction logic.
- Documents instruction evolution with version-controlled `INSTRUCTION_LOG.md`.

## Step-by-Step Instructions

1. **Establish a baseline** by running the eval suite against the current
   instruction and recording all metric scores.

2. **Identify the failure type** from metrics:
   - `tool_trajectory_avg_score < 1.0` → Agent calls wrong tool, skips tool,
     or calls in wrong order.
   - `response_match_score < 0.8` → Output format or content is off.
   - Agent ignores all tools → Instruction doesn't mention tools clearly.

3. **Inspect the failing eval case** using ADK Web UI Trace View to see:
   - What model request was sent (the exact system prompt + conversation).
   - What decision the model made (which tool it chose, or if it skipped tools).

4. **Formulate a hypothesis** about the instruction's shortcoming:
   - "The instruction doesn't tell the agent to always search before quoting."
   - "The instruction doesn't specify the output list format."

5. **Make a targeted revision** — change one thing per iteration:
   ```python
   # Before:
   instruction = "Help users book flights."

   # After (adds explicit tool call sequence):
   instruction = """
   Help users book flights.
   STEP 1: Always call search_flights first before answering any price query.
   STEP 2: Present results in a numbered list.
   STEP 3: Confirm with user before calling process_payment.
   """
   ```

6. **Re-run the eval suite** and compare scores to the baseline.

7. **Accept the change** if metrics improve. Revert if they don't or if other
   metrics regress.

8. **Document the change** in `INSTRUCTION_LOG.md`:
   ```markdown
   ## v1.2 — 2024-03-15
   **Change**: Added explicit 3-step tool call sequence.
   **Reason**: tool_trajectory_avg_score was 0.5 (skip of search_flights).
   **Result**: tool_trajectory_avg_score improved to 1.0 (10 eval cases).
   ```

9. **Repeat** until all metrics meet thresholds.

## Input Format

```python
{
  "agent_goal": "Flight booking with correct tool sequence",
  "user_input": "How much does a flight to London cost?",
  "context": {"booking_step": "search"},
  "tools": ["search_flights", "process_payment"],
  "output_format": "numbered list",
  "constraints": [
    "Must call search_flights before quoting prices",
    "Must confirm before payment"
  ]
}
```

## Output Format

```python
{
  "system_prompt": "Help users book flights.\nSTEP 1: Always call search_flights...",
  "instructions": "...",
  "expected_output_format": {"type": "numbered_list"},
  "tool_usage_guidance": {
    "search_flights": "Call FIRST for any price or availability query",
    "process_payment": "Call ONLY after user confirms with 'yes'"
  }
}
```

## Error Handling

- **Instruction change causes regression in other metrics**: Revert the last
  change. The instruction is a holistic document — changes can have unexpected
  effects on adjacent behaviors.
- **No metric improvement after 3 iterations**: The problem may be in the tool
  docstring, not the instruction. Try improving the tool `Args` and `Returns`
  docstring sections.
- **Conflicting instructions** (new clause contradicts existing): Audit the
  full instruction for contradictions before adding new clauses.
- **Instruction too long** (> 2000 tokens after tuning): Extract stable sections
  into tool docstrings or a shared `InstructionProvider` helper function.
- **Agent ignores explicit `MUST` directives**: Try rephrasing as a numbered
  step sequence instead of a directive — models respond better to procedural
  instructions.

## Examples

### Example 1 — Iterative tuning with eval feedback

```python
import asyncio
from google.adk.agents import LlmAgent
from google.adk.evaluation.agent_evaluator import AgentEvaluator

# v1: Initial instruction
V1_INSTRUCTION = "You are a flight booking assistant. Help users find and book flights."

# After eval: tool_trajectory_avg_score = 0.5 (agent skips search_flights)
# Hypothesis: Instruction doesn't enforce tool call sequence.

# v2: Add explicit tool call sequence
V2_INSTRUCTION = """
You are a flight booking assistant for AcmeAir.

REQUIRED SEQUENCE:
1. When the user asks about flights or prices, call search_flights FIRST.
2. Present results as a numbered list.
3. Ask the user to confirm their selection.
4. Only then call process_payment.

Never quote prices from memory — always use search_flights results.
"""

async def compare_instructions():
    for version, instruction in [("v1", V1_INSTRUCTION), ("v2", V2_INSTRUCTION)]:
        from google.adk.agents import LlmAgent
        agent = LlmAgent(
            name=f"flight_agent_{version}",
            model="gemini-2.0-flash",
            instruction=instruction,
        )
        print(f"\n--- Evaluating {version} ---")
        try:
            await AgentEvaluator.evaluate(
                agent_module="flight_agent",
                eval_dataset_file_path_or_dir="tests/eval/flight_booking.test.json",
            )
            print(f"{version}: PASS")
        except AssertionError as e:
            print(f"{version}: FAIL — {e}")
```

### Example 2 — Context-conditional instruction via InstructionProvider

```python
from google.adk.agents import LlmAgent
from google.adk.agents.readonly_context import ReadonlyContext

STEP_INSTRUCTIONS = {
    "search": "Your current task: help the user find available flights using search_flights.",
    "confirm": "Your current task: present the selected flight clearly and ask for confirmation.",
    "payment": "Your current task: call process_payment ONLY after the user says 'yes'.",
}

def adaptive_instruction(ctx: ReadonlyContext) -> str:
    """Applies step-specific instruction tuning based on booking workflow state."""
    step = ctx.state.get("booking_step", "search")
    step_guide = STEP_INSTRUCTIONS.get(step, STEP_INSTRUCTIONS["search"])
    return f"""
You are a flight booking assistant for AcmeAir.
{step_guide}

GLOBAL RULES:
- Never invent flight data — always use tool results.
- Present flights as a numbered list with airline, time, duration, price.
- Decline all off-topic questions politely.
"""

agent = LlmAgent(
    name="adaptive_flight_agent",
    model="gemini-2.0-flash",
    instruction=adaptive_instruction,
    tools=[search_flights, process_payment],
)
```

### Example 3 — Tool docstring improvement as instruction tuning

```python
def search_flights(origin: str, destination: str, date: str, tool_context) -> dict:
    """Searches for available flights between two airports on a given date.

    WHEN TO CALL: Call this tool BEFORE quoting ANY price or availability.
    Do NOT ask the user for more information before calling this tool —
    the origin, destination, and date are sufficient to search.

    Args:
        origin (str): IATA airport code for the departure airport, e.g. 'JFK'.
        destination (str): IATA airport code for the arrival airport, e.g. 'LHR'.
        date (str): Travel date in ISO 8601 format, e.g. '2024-03-15'.
        tool_context: ADK tool context.

    Returns:
        dict: Contains 'flights' (list of flight dicts) and 'search_id' (str).
    """
    # ... implementation ...
    return {"flights": [], "search_id": "srch_001"}
```

## Edge Cases

- **Instruction improvement stalls** (no metric improvement after 5 iterations):
  Escalate to human-in-the-loop review. The ground-truth eval cases may be
  misaligned with the actual correct behavior.
- **Different models respond differently to the same instruction**: Tune
  instructions per model. A `gemini-2.0-pro` instruction may not work as well
  with `gemini-2.0-flash`.
- **Instruction regression after model upgrade**: Re-run the full eval suite
  whenever the model is upgraded. Store the instruction version in `meta:` state.
- **Over-tuning** (instruction becomes too rigid): Leave some room for the LLM
  to reason. Over-constrained instructions lead to brittle agents that fail on
  edge cases.

## Integration Notes

- **`google-adk-automated-evaluation`**: The eval pipeline is the feedback loop
  for instruction tuning. Never tune without a quantitative evaluation.
- **`google-adk-error-analysis`**: Error analysis identifies which specific
  behavior needs instruction correction.
- **`google-adk-system-prompts`**: Instruction tuning refines the system prompt
  created by `system-prompts` skill.
- **ADK Web UI Trace View**: Use to inspect the exact model request (system
  prompt + conversation) that led to a bad decision.
