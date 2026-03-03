---
name: tool-validation
description: >
  Use this skill to validate tool inputs and outputs against schemas, business
  rules, and safety constraints before or after tool execution. Distinct from
  structured-tool-inputs: this skill covers pre-call guards, post-call
  assertions, and business-logic validation beyond schema conformance.
compatibility:
  - Claude Code
  - Google ADK
  - OpenAI Agents
  - MCP-based agents
  - Tool-using LLM agents
metadata:
  author: agent-skills-generator
  version: "1.0"
---

## Purpose

This skill implements validation layers for tool inputs (pre-execution guards)
and tool outputs (post-execution assertions). It catches business-rule
violations, safety issues, and semantic errors that schema validation alone
cannot detect — such as a date range where start > end, or a query that
contains prompt-injection patterns.

## When to Use

- When tool inputs require business-logic validation beyond type checking.
- When tool outputs must conform to expected ranges or formats before being
  passed to the LLM.
- When implementing safety guardrails on tool arguments (e.g., blocking
  dangerous shell commands).
- When enforcing domain-specific constraints (e.g., amount > 0, date in future).

## When NOT to Use

- Do not use for basic type coercion — use `structured-tool-inputs` for that.
- Do not use for error handling of exceptions — use `tool-error-handling`.
- Do not use to validate agent instructions or prompts.

## Capabilities

- Pre-call validation: Checks inputs before the tool function executes.
- Post-call validation: Checks outputs before returning to the LLM.
- Business rule validation: Enforces domain constraints.
- Safety validation: Detects and blocks dangerous or malicious inputs.
- Returns structured validation error dicts compatible with the agent loop.

## Step-by-Step Instructions

1. **Identify validation rules** for each tool parameter:
   - Range constraints: `amount > 0 and amount <= 10000`
   - Format constraints: `date matches 'YYYY-MM-DD'`
   - Relational constraints: `start_date < end_date`
   - Safety constraints: `query does not contain shell metacharacters`

2. **Create a pre-call validator function** named `validate_<tool_name>_input`:
   ```python
   def validate_get_report_input(start_date: str, end_date: str, metric: str) -> dict | None:
       """Returns None if valid, or error dict if invalid."""
       ...
   ```

3. **Return `None` when valid** (no error). Return an error dict when invalid:
   ```python
   return {"error": "start_date must be before end_date.", "error_type": "invalid_input", "recoverable": False}
   ```

4. **Call the validator before tool execution** in the agent loop or decorator:
   ```python
   validation_error = validate_get_report_input(**tool_args)
   if validation_error:
       return validation_error
   result = get_report(**tool_args)
   ```

5. **Create a post-call validator** named `validate_<tool_name>_output` when
   outputs must be checked:
   ```python
   def validate_search_output(result: dict) -> dict | None:
       if result.get("count", 0) < 0:
           return {"error": "Search returned negative count — API bug.", ...}
       return None
   ```

6. **Apply post-call validation** and return the error if validation fails.

7. **Log all validation failures** at WARNING level with the failed values.

## Input Format

Pre-call:
```
tool_name: <str>
tool_args: dict
validation_rules: list[Callable]  # Each returns None or error dict
```

Post-call:
```
tool_name: <str>
tool_result: dict
output_rules: list[Callable]
```

## Output Format

If valid: `None` (no error, proceed)

If invalid:
```python
{
  "error": "start_date '2024-12-31' must be before end_date '2024-01-01'.",
  "error_type": "invalid_input",
  "recoverable": False,
  "field": "start_date"
}
```

## Error Handling

- Validators must NOT raise exceptions — they must return error dicts.
- If a validator itself throws (a bug in validation logic), catch it and return:
  `{"error": "Validation system error.", "error_type": "unexpected", "recoverable": False}`
- Do not apply multiple validators and return only one error — return all
  failed validations as a list.

## Examples

### Example 1 — Pre-call business rule validation

```python
import re
from datetime import datetime

def validate_get_report_input(start_date: str, end_date: str, metric: str) -> dict | None:
    errors = []

    # Date format validation
    date_fmt = "%Y-%m-%d"
    try:
        start = datetime.strptime(start_date, date_fmt)
        end = datetime.strptime(end_date, date_fmt)
    except ValueError as e:
        errors.append({"field": "date", "message": f"Invalid date format: {e}"})
        return {"error": "Validation failed", "details": errors, "error_type": "invalid_input", "recoverable": False}

    # Relational constraint
    if start >= end:
        errors.append({"field": "start_date", "message": "start_date must be before end_date."})

    # Allowed values
    allowed_metrics = ["revenue", "users", "sessions", "clicks"]
    if metric not in allowed_metrics:
        errors.append({"field": "metric", "message": f"metric must be one of {allowed_metrics}"})

    if errors:
        return {"error": "Validation failed", "details": errors, "error_type": "invalid_input", "recoverable": False}
    return None
```

### Example 2 — Safety validation (prompt injection guard)

```python
DANGEROUS_PATTERNS = [r";\s*rm\s+-rf", r"\|\s*bash", r"`.*`", r"\$\(.*\)"]

def validate_shell_command_input(command: str) -> dict | None:
    for pattern in DANGEROUS_PATTERNS:
        if re.search(pattern, command):
            return {
                "error": f"Command contains a blocked pattern: '{pattern}'.",
                "error_type": "invalid_input",
                "recoverable": False,
            }
    return None
```

### Example 3 — Post-call output validation

```python
def validate_search_output(result: dict) -> dict | None:
    if "results" not in result:
        return {"error": "Search result missing 'results' field.", "error_type": "unexpected", "recoverable": False}
    if not isinstance(result["results"], list):
        return {"error": "'results' must be a list.", "error_type": "unexpected", "recoverable": False}
    return None
```

## Edge Cases

- **LLM attempts prompt injection via tool args**: Apply safety validation on
  all string inputs that will be used in downstream system calls.
- **Validation is too strict and blocks valid inputs**: Log all blocked inputs
  and review periodically. Do not over-constrain.
- **Output validation fails on a rare API response shape**: Treat as
  `unexpected` error type and escalate for manual review.

## Integration Notes

- **Google ADK**: Use `before_tool_callback` and `after_tool_callback` hooks on
  `LlmAgent` to apply validation without modifying tool functions.
- **OpenAI Agents SDK**: Use `input_guardrails` and `output_guardrails` on
  `Agent` config for built-in guardrail support.
- **structured-tool-inputs skill**: Handles schema/type validation. This skill
  adds business logic and safety validation on top of it.
- **tool-error-handling skill**: Use the same error dict format for consistency
  with the agent loop's error processing.
