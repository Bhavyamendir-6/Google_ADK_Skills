---
name: custom-tool-development
description: >
  Use this skill when developing a net-new, fully custom tool from scratch that
  does not fit existing categories (API, database, web). Covers the complete
  lifecycle: design, implementation, testing, documentation, and integration
  into a production agent system.
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

This skill provides the end-to-end process for designing and implementing a
custom tool that extends an agent's capabilities with bespoke logic not covered
by standard API, database, or web patterns. It enforces a consistent development
lifecycle: requirements → design → implementation → testing → registration →
documentation.

## When to Use

- When creating a tool that generates content (e.g., PDFs, images, code).
- When wrapping local system utilities (e.g., file I/O, subprocess execution).
- When building domain-specific data processing tools (e.g., parsing invoices,
  computing financial metrics).
- When no existing tool pattern covers the required functionality.

## When NOT to Use

- Do not use when the tool wraps an HTTP API (see `external-api-tools`).
- Do not use when the tool queries a database (see `database-tools`).
- Do not use when the tool fetches web content (see `web-tools`).
- Do not use to modify existing tools — only for net-new development.

## Capabilities

- Provides a development checklist for custom tool lifecycle.
- Defines the tool contract (inputs, outputs, errors) before implementation.
- Implements the tool with appropriate error handling and logging.
- Generates the tool's unit test suite.
- Documents the tool for LLM consumption and human maintainers.
- Integrates the finished tool into an agent configuration.

## Step-by-Step Instructions

### Phase 1: Requirements

1. **Write a one-sentence purpose statement** for the tool.

2. **List all inputs** with name, type, description, and whether required.

3. **Define the output structure** — what keys the result dict must always contain.

4. **List all error conditions** and expected error types (`transient`, `permanent`,
   `invalid_input`).

5. **Identify side effects** (writes to file system, sends notifications, etc.)
   and mark the tool accordingly in its docstring.

### Phase 2: Design

6. **Choose the implementation approach**:
   - Pure Python: For computation, data transformation, format conversion.
   - Subprocess: For CLI tools, system utilities. Use `subprocess.run` with
     `shell=False` and explicit argument lists.
   - SDK/library: For wrapping third-party Python SDKs.

7. **Design the output schema** as a typed dict or Pydantic model.

8. **Identify reusable components** from existing skills to avoid duplication:
   - Apply `@tool_error_handler` from `tool-error-handling`.
   - Apply `@with_retries` from `tool-retries` if transient failures are possible.
   - Apply input validation from `tool-validation` if needed.

### Phase 3: Implementation

9. **Write the tool function** following `tool-definition` conventions:
   - Descriptive snake_case name.
   - Full type annotations.
   - Google-style docstring with Args, Returns, Raises, Note (if side effects).

10. **Implement the function body** in stages:
    - Input preprocessing.
    - Core logic.
    - Result normalization.
    - Return result dict.

11. **Add `@tool_error_handler`** as the innermost decorator (directly on the function).

12. **Add `@with_retries`** as the outermost decorator if applicable.

### Phase 4: Testing

13. **Write unit tests** for:
    - Happy path with valid inputs.
    - Each error condition.
    - Edge cases (empty input, boundary values, large inputs).

14. **Run tests** and confirm all pass before integration.

### Phase 5: Registration and Documentation

15. **Register the tool** with the agent following `tool-registration`.

16. **Run a smoke test** with a real agent prompt that triggers the tool.

17. **Update the tool's module docstring** with usage examples and the
    frameworks it has been tested with.

## Input Format

```
tool_purpose: <str>           # One sentence
tool_name: <snake_case str>
parameters:
  - name: <str>
    type: <python_type>
    required: bool
    description: <str>
output_fields:
  - name: <str>
    type: <python_type>
    description: <str>
side_effects: bool
approach: pure_python | subprocess | sdk_library
```

## Output Format

The tool function following all conventions:

```python
@with_retries(max_attempts=3)     # Only if transient failures possible
@tool_error_handler
def tool_name(param1: str, param2: int) -> dict:
    """One-line purpose statement.

    Note: This tool has the following side effects: <describe>.

    Args:
        param1 (str): Description.
        param2 (int): Description.

    Returns:
        dict: Contains 'output_field_1' (type) and 'output_field_2' (type).

    Raises:
        ValueError: If param1 is invalid.
    """
    # Phase 1: Input preprocessing
    ...
    # Phase 2: Core logic
    ...
    # Phase 3: Result normalization
    return {"output_field_1": ..., "output_field_2": ...}
```

## Error Handling

- All exceptions are handled by `@tool_error_handler`.
- Raise domain-specific exceptions from within the function body (e.g.,
  `ValueError`, `FileNotFoundError`) — the decorator catches and classifies them.
- For subprocess tools: check `result.returncode != 0` and raise `RuntimeError`
  with `result.stderr`.
- For SDK tools: map SDK-specific exceptions to standard Python exceptions before
  raising.

## Examples

### Example 1 — File generation tool

```python
import os
import hashlib
from pathlib import Path

@tool_error_handler
def generate_csv_report(data: list[dict], filename: str, output_dir: str = "/tmp") -> dict:
    """Generates a CSV file from a list of row dicts and returns the file path.

    Note: This tool writes a file to the local filesystem.

    Args:
        data (list[dict]): Rows to write. All dicts must have the same keys.
        filename (str): Name for the output file (without extension).
        output_dir (str): Directory to write to. Defaults to '/tmp'.

    Returns:
        dict: Contains 'file_path' (str), 'row_count' (int), and 'size_bytes' (int).

    Raises:
        ValueError: If data is empty or rows have inconsistent keys.
    """
    import csv

    if not data:
        raise ValueError("data must not be empty")

    headers = list(data[0].keys())
    if not all(set(row.keys()) == set(headers) for row in data):
        raise ValueError("All rows must have the same keys")

    safe_name = "".join(c if c.isalnum() or c in "-_" else "_" for c in filename)
    file_path = Path(output_dir) / f"{safe_name}.csv"

    with open(file_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)

    size = file_path.stat().st_size
    return {"file_path": str(file_path), "row_count": len(data), "size_bytes": size}
```

### Example 2 — Subprocess-based tool

```python
import subprocess

@tool_error_handler
def run_python_linter(file_path: str) -> dict:
    """Runs flake8 linter on a Python file and returns linting results.

    Args:
        file_path (str): Absolute path to the Python file to lint.

    Returns:
        dict: Contains 'passed' (bool), 'issues' (list of str), and 'issue_count' (int).

    Raises:
        FileNotFoundError: If file_path does not exist.
        RuntimeError: If flake8 is not installed.
    """
    if not os.path.isfile(file_path):
        raise FileNotFoundError(f"File not found: {file_path}")

    result = subprocess.run(
        ["flake8", "--max-line-length=100", file_path],
        capture_output=True,
        text=True,
        timeout=30,
        shell=False,
    )
    issues = [line for line in result.stdout.splitlines() if line.strip()]
    return {
        "passed": len(issues) == 0,
        "issues": issues[:50],  # cap at 50 issues
        "issue_count": len(issues),
    }
```

### Example 3 — Unit test template

```python
import pytest
from my_tools import generate_csv_report

def test_generate_csv_report_happy_path():
    result = generate_csv_report(data=[{"a": 1, "b": 2}], filename="test")
    assert "file_path" in result
    assert result["row_count"] == 1
    assert result["size_bytes"] > 0

def test_generate_csv_report_empty_data():
    result = generate_csv_report(data=[], filename="test")
    assert "error" in result
    assert result["error_type"] == "invalid_input"

def test_generate_csv_report_inconsistent_keys():
    result = generate_csv_report(data=[{"a": 1}, {"b": 2}], filename="test")
    assert "error" in result
```

## Edge Cases

- **Tool requires external binary**: Document the binary dependency in the
  module docstring and in `requirements.txt` or `pyproject.toml`.
- **Tool produces large output files**: Return the file path only; never embed
  binary content in the result dict.
- **Tool must be async**: Use `async def` and `await` throughout; mark in
  docstring as `Note: This tool is async.`
- **Tool has global mutable state**: Avoid it. Use dependency injection or
  module-level constants for configuration.
- **Tool needs user approval before side effects**: Add a `dry_run: bool = True`
  parameter. Execute side effects only when `dry_run=False`.

## Integration Notes

- **Google ADK**: After creating the tool, register it in `LlmAgent(tools=[...])`.
  Use `before_tool_callback` to intercept calls for logging or approval flows.
- **skill composition**: Custom tools should import and apply `@tool_error_handler`
  from `tool-error-handling`, `@with_retries` from `tool-retries`, and validators
  from `tool-validation` as needed.
- **MCP**: Custom tools can be deployed as MCP servers using `FastMCP`. Apply
  the same function conventions and decorate with `@mcp.tool()`.
- **OpenAI Agents SDK**: Pass the custom tool function directly to
  `Agent(tools=[custom_tool])`. The SDK wraps it automatically.
