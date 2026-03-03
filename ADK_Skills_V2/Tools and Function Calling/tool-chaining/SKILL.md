---
name: tool-chaining
description: >
  Use this skill when an agent must invoke multiple tools in sequence, where
  the output of one tool becomes the input of the next. Covers dependency
  resolution, data passing between tools, and designing multi-step tool
  pipelines within an agent turn.
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

This skill governs the design and execution of tool chains — sequences of tool
calls where each step depends on the result of the previous one. It ensures
the agent correctly passes data through multi-tool pipelines, handles
intermediate failures, and maintains context between steps.

## When to Use

- When a task requires more than one tool call to complete.
- When tool B needs data produced by tool A.
- When designing agent pipelines (e.g., search → summarize → send).
- When implementing sequential agent workflows in Google ADK or LangChain.

## When NOT to Use

- Do not use when tools are independent and can run in parallel.
- Do not use for single-tool workflows.
- Do not use to compose agents (see multi-agent architecture patterns).

## Capabilities

- Sequences tool calls with explicit data passing between steps.
- Extracts specific fields from tool outputs to pass as inputs to next tools.
- Implements conditional branching (execute tool C only if tool B succeeded).
- Logs each step's input and output for observability.
- Handles intermediate failures without corrupting downstream steps.

## Step-by-Step Instructions

1. **Map the task into a tool dependency graph**:
   ```
   Tool A → Tool B → Tool C
   ```
   Write this explicitly before coding. Identify which output fields of A
   map to which input fields of B.

2. **Define each tool** independently following the `tool-definition` skill.
   Tools must not be aware of each other.

3. **Write the chain orchestrator function** that:
   a. Calls Tool A with initial inputs.
   b. Extracts the required field(s) from Tool A's result.
   c. Passes extracted field(s) as arguments to Tool B.
   d. Repeats for each step.

4. **Validate each intermediate result** before passing to the next tool:
   - Check for `"error"` key in result dict.
   - If present, abort the chain and return the error.

5. **Return a final result dict** that includes the output of the last tool
   and, optionally, a summary of all intermediate outputs.

6. **For LLM-driven chains**: Let the model decide the sequence by providing
   all tools and a task description. The model will emit tool calls in order.
   Implement the `function-calling` loop to handle each turn.

7. **For deterministic chains**: Implement the orchestrator in Python; the LLM
   is not involved in sequencing.

## Input Format

```
task: <str>               # High-level task description
initial_inputs: dict      # Inputs for the first tool in the chain
chain: list[str]          # Ordered list of tool names to execute
field_mappings: list[dict]  # [{from_tool: str, from_field: str, to_tool: str, to_field: str}]
```

## Output Format

```python
{
  "chain_result": <final_tool_output>,
  "steps": [
    {"tool": "tool_a", "input": {...}, "output": {...}},
    {"tool": "tool_b", "input": {...}, "output": {...}},
  ],
  "success": True
}
```

On failure:
```python
{
  "chain_result": None,
  "steps": [...],  # Steps completed before failure
  "success": False,
  "failed_at": "tool_b",
  "error": "..."
}
```

## Error Handling

- If any tool in the chain returns `{"error": ...}`, stop the chain immediately.
- Return `success: False` and `failed_at: <tool_name>`.
- Do NOT pass error results as inputs to subsequent tools.
- Log all intermediate steps regardless of success or failure.

## Examples

### Example 1 — Deterministic Python chain

```python
def research_and_summarize(query: str) -> dict:
    """Chain: search_web → extract_key_points → format_report"""
    steps = []

    # Step 1: Search
    search_result = search_web(query=query)
    steps.append({"tool": "search_web", "input": {"query": query}, "output": search_result})
    if "error" in search_result:
        return {"success": False, "failed_at": "search_web", "error": search_result["error"], "steps": steps}

    # Step 2: Extract key points
    extract_result = extract_key_points(text=search_result["content"])
    steps.append({"tool": "extract_key_points", "input": {"text": search_result["content"]}, "output": extract_result})
    if "error" in extract_result:
        return {"success": False, "failed_at": "extract_key_points", "error": extract_result["error"], "steps": steps}

    # Step 3: Format report
    report = format_report(points=extract_result["key_points"], title=query)
    steps.append({"tool": "format_report", "input": {"points": extract_result["key_points"], "title": query}, "output": report})

    return {"success": True, "chain_result": report, "steps": steps}
```

### Example 2 — LLM-driven chain with function-calling loop

```python
# Register all chain tools and run the function-calling loop.
# The LLM sequences the calls automatically given the task prompt.
agent = LlmAgent(
    name="research_agent",
    model="gemini-2.0-flash",
    tools=[search_web, extract_key_points, format_report],
    instruction=(
        "To answer research questions: first search the web, "
        "then extract key points, then format a report."
    ),
)
```

## Edge Cases

- **Circular dependency**: Never design chain A → B → A. Detect cycles in the
  dependency graph at design time.
- **Tool output too large for next tool's input**: Truncate or summarize the
  output before passing. Document the truncation in the chain log.
- **Optional steps in chain**: Use `if condition: result = tool_c(...)` and
  skip steps that are not needed based on previous results.
- **Parallel sub-steps**: If steps B and C both depend on A but not on each
  other, run them concurrently using `asyncio.gather`.

## Integration Notes

- **Google ADK Sequential Agents**: Use `SequentialAgent` with sub-agents, each
  wrapping a tool, for managed sequential pipelines.
- **LangChain**: Use `RunnableSequence` or LCEL pipe syntax for declarative
  chains.
- **OpenAI Agents**: Use `handoff` or multi-turn loops; the SDK does not natively
  enforce tool ordering.
- **tool-error-handling skill**: Apply at each step in the chain to handle
  failures without breaking the pipeline.
