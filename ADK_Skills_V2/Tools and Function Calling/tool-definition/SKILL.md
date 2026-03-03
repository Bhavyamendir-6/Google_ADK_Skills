---
name: tool-definition
description: >
  Use this skill when you need to define a tool (function) that an LLM agent
  can call. Covers writing the function signature, docstring, type annotations,
  and parameter descriptions so that the LLM correctly understands the tool's
  purpose, inputs, and outputs.
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

This skill governs how to write a well-formed tool definition that an LLM agent
runtime can introspect to construct a correct function call. A tool definition
includes the function signature, type annotations, and a structured docstring
that describes each parameter and return value clearly.

## When to Use

- When creating a new Python/TypeScript/Go function intended for use as an agent tool.
- When an existing function needs to be updated to be discoverable and callable by an LLM.
- When on-boarding a tool to Google ADK, OpenAI function calling, or MCP.

## When NOT to Use

- Do not use when you only need to register an already-defined tool (see `tool-registration`).
- Do not use for defining agents themselves—only tools (callable functions).
- Do not apply to class methods that are not meant to be exposed to the LLM.

## Capabilities

- Produces a correctly typed Python function with a Google-style docstring.
- Ensures all parameters have explicit type annotations.
- Ensures the return type is explicit and documented.
- Generates parameter descriptions consumable by LLM schema extraction.
- Handles optional parameters with default values.

## Step-by-Step Instructions

1. **Identify the tool's single responsibility.** Each tool must do exactly one
   thing. If the function does multiple things, split it.

2. **Choose a descriptive snake_case function name** (max 64 chars) that
   unambiguously describes the action: `get_weather`, `search_products`,
   `send_email`.

3. **Annotate every parameter** with a concrete Python type:
   - Use `str`, `int`, `float`, `bool` for primitives.
   - Use `list[str]`, `dict[str, Any]` for collections.
   - Use `Optional[T]` (or `T | None`) only when the parameter is truly optional.

4. **Annotate the return type** explicitly. Prefer primitive or typed dict
   returns for LLM-parseable results.

5. **Write a Google-style docstring** with:
   - A one-line summary sentence.
   - `Args:` section listing every parameter with its type and description.
   - `Returns:` section describing the return value type and structure.
   - `Raises:` section if the function may raise exceptions.

6. **Keep the function body minimal and focused.** The definition is the
   contract; implementation details do not affect LLM behavior.

7. **Validate your definition** by running schema extraction mentally:
   - Every parameter appears in `Args:`.
   - No parameter is undocumented.
   - The summary sentence alone is sufficient to understand when to call this tool.

## Input Format

The agent will receive a natural-language request that implies a tool should be
created or updated. Extract:

```
tool_name: <snake_case name>
purpose: <one sentence>
parameters:
  - name: <param_name>
    type: <python_type>
    required: true|false
    description: <what this param controls>
return_type: <python_type>
return_description: <what is returned>
```

## Output Format

A complete Python function definition:

```python
def tool_name(param1: str, param2: int, param3: Optional[str] = None) -> dict:
    """One-line summary of what this tool does.

    Args:
        param1 (str): Description of param1.
        param2 (int): Description of param2.
        param3 (Optional[str]): Description of param3. Defaults to None.

    Returns:
        dict: A dictionary containing keys 'result' (str) and 'status' (str).

    Raises:
        ValueError: If param1 is empty.
    """
    ...
```

## Error Handling

- If the tool purpose is ambiguous, stop and request clarification before
  writing the definition.
- If a parameter type cannot be determined, default to `str` and add a comment
  `# TODO: confirm type`.
- If the function has side effects (writes to DB, calls external API), add a
  `Note:` section in the docstring stating this explicitly.

## Examples

### Example 1 — Simple lookup tool

```python
def get_stock_price(ticker: str, currency: str = "USD") -> dict:
    """Retrieves the current stock price for a given ticker symbol.

    Args:
        ticker (str): The stock ticker symbol, e.g., 'AAPL'.
        currency (str): The currency for the price. Defaults to 'USD'.

    Returns:
        dict: Contains 'ticker' (str), 'price' (float), and 'currency' (str).

    Raises:
        ValueError: If ticker is empty or invalid.
    """
    ...
```

### Example 2 — Write tool with side effects

```python
def send_slack_message(channel: str, message: str, mention_user: Optional[str] = None) -> dict:
    """Sends a message to a Slack channel.

    Note: This tool has a side effect of posting a message to Slack.

    Args:
        channel (str): The Slack channel ID, e.g., 'C01ABCD1234'.
        message (str): The message text to send.
        mention_user (Optional[str]): Slack user ID to @mention. Defaults to None.

    Returns:
        dict: Contains 'ok' (bool) and 'ts' (str) — the message timestamp.

    Raises:
        SlackApiError: If the API call fails.
    """
    ...
```

## Edge Cases

- **No-parameter tools**: Add `-> str` return type and a clear docstring. Do
  not omit the return annotation.
- **Tools returning None**: Annotate as `-> None` only if the tool is purely
  for side effects. Prefer returning a status dict instead.
- **Variable-length arguments**: Avoid `*args` and `**kwargs`—LLMs cannot
  reason about them. Use explicit named parameters.
- **Large return payloads**: If the return value can be very large, document
  the maximum expected structure and truncation behavior in the docstring.

## Integration Notes

- **Google ADK**: ADK uses the docstring's `Args:` block to generate the JSON
  schema passed to the model. Missing arg descriptions = missing schema fields.
- **OpenAI function calling**: The `description` field in the JSON schema maps
  directly to the function's one-line docstring summary.
- **MCP**: Tool definitions must align with `inputSchema` JSON Schema; the
  docstring drives automatic schema generation.
- **Claude Code**: Claude reads the docstring directly during tool invocation.
  Use precise, unambiguous language.
