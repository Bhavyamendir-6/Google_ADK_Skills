---
name: function-calling
description: >
  Use this skill when an LLM agent needs to decide which registered tool to
  invoke, construct the correct arguments, execute the call, and feed the result
  back into the model context. Covers the full request-invoke-respond cycle for
  LLM function calling.
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

This skill governs the complete function-calling loop: from the LLM emitting a
tool-call request, through argument extraction and validation, to function
execution and result injection back into the model context. It ensures the
agent correctly manages the tool-call lifecycle without data loss or misrouting.

## When to Use

- When implementing the agent loop that processes LLM tool-call responses.
- When debugging why a tool invocation produces incorrect arguments.
- When the LLM calls the wrong tool or fails to call any tool.
- When adding function-calling support to a custom agent runner.

## When NOT to Use

- Do not use for defining the tool itself (see `tool-definition`).
- Do not use for registering tools with the agent (see `tool-registration`).
- Do not use when the issue is in the tool's internal logic, not the calling
  mechanism.

## Capabilities

- Implements the full tool-call event loop for Google ADK, OpenAI, and MCP.
- Extracts tool name and arguments from model responses.
- Dispatches to the correct function by name.
- Injects tool results back into the conversation context.
- Handles parallel tool calls (multiple tools called in a single turn).

## Step-by-Step Instructions

1. **Run the model** with the user message and the registered tools list.

2. **Inspect the model response** for tool-call events:
   - Google ADK: Check for `FunctionCall` parts in `response.candidates[0].content.parts`.
   - OpenAI: Check `response.choices[0].message.tool_calls`.

3. **For each tool call** in the response:
   a. Extract `tool_name` (string) and `tool_args` (dict).
   b. Look up the function in the registered tools map by `tool_name`.
   c. If the function is not found, return an error result (see Error Handling).

4. **Execute the function** with the extracted arguments:
   ```python
   result = registered_tools[tool_name](**tool_args)
   ```

5. **Serialize the result** to a string or JSON string if not already a string.

6. **Append the tool result** to the conversation context:
   - Google ADK: Add a `FunctionResponse` part to a new `Content` with `role="tool"`.
   - OpenAI: Add a message with `role="tool"`, `tool_call_id`, and `content`.

7. **Re-run the model** with the updated context to get the final response.

8. **Repeat** steps 2–7 until the model response contains no tool calls
   (i.e., it returns a plain text response).

## Input Format

```
user_message: <str>
registered_tools: dict[str, Callable]  # {function_name: function_object}
model: <model_identifier>
conversation_history: list[dict]  # existing messages
```

## Output Format

```
final_response: <str>          # The model's text answer after all tool calls
tool_calls_made: list[dict]    # Log of each tool call: {name, args, result}
turn_count: int                # Number of model turns taken
```

## Error Handling

- **Unknown tool name**: Return `{"error": "Tool '{name}' not found."}` as the
  tool result and continue the loop. Do NOT raise an exception that breaks the loop.
- **Missing required argument**: Return `{"error": "Missing required argument '{arg}'."}`.
- **Tool raises an exception**: Catch all exceptions, return
  `{"error": str(exception)}` as the tool result, and continue the loop.
- **Infinite loop guard**: Set a maximum turn count (default: 10). If exceeded,
  abort and return the last partial response with a warning.
- **Empty tool_calls list**: Treat as final response — no re-run needed.

## Examples

### Example 1 — Google ADK manual function-call loop

```python
import json
from google.genai.types import Content, Part, FunctionResponse

def run_agent_loop(agent, user_message: str, tools: dict) -> str:
    contents = [Content(role="user", parts=[Part(text=user_message)])]
    max_turns = 10

    for _ in range(max_turns):
        response = agent.model.generate_content(contents=contents, tools=list(tools.values()))
        candidate = response.candidates[0]
        function_calls = [p for p in candidate.content.parts if p.function_call]

        if not function_calls:
            return "".join(p.text for p in candidate.content.parts if p.text)

        contents.append(candidate.content)
        tool_results = []

        for fc in function_calls:
            fn = tools.get(fc.function_call.name)
            if fn is None:
                result = {"error": f"Tool '{fc.function_call.name}' not found."}
            else:
                try:
                    result = fn(**fc.function_call.args)
                except Exception as e:
                    result = {"error": str(e)}

            tool_results.append(
                Part(function_response=FunctionResponse(
                    name=fc.function_call.name,
                    response=result,
                ))
            )

        contents.append(Content(role="tool", parts=tool_results))

    return "Max turns exceeded."
```

### Example 2 — OpenAI function-calling loop

```python
import json
from openai import OpenAI

client = OpenAI()

def run_openai_loop(messages: list, tools_schema: list, dispatch: dict) -> str:
    for _ in range(10):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools_schema,
            tool_choice="auto",
        )
        msg = response.choices[0].message
        if not msg.tool_calls:
            return msg.content

        messages.append(msg)
        for tc in msg.tool_calls:
            fn = dispatch.get(tc.function.name)
            args = json.loads(tc.function.arguments)
            try:
                result = fn(**args) if fn else {"error": "Unknown tool"}
            except Exception as e:
                result = {"error": str(e)}
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": json.dumps(result),
            })

    return "Max turns exceeded."
```

## Edge Cases

- **Parallel tool calls**: Process all tool calls in one turn before re-running
  the model. Do NOT run the model after each individual tool result.
- **Tool call with no arguments**: Pass an empty dict `{}` to the function.
  Do NOT skip execution.
- **Model hallucinates argument values**: Validate inputs with `tool-validation`
  skill before executing.
- **Streaming responses**: Buffer streamed chunks and reconstruct the tool-call
  delta before dispatching.

## Integration Notes

- **Google ADK**: ADK's `Runner` handles this loop automatically. Only implement it
  manually when building a custom runner outside of ADK.
- **OpenAI Agents SDK**: The SDK's `Runner.run()` handles the loop; use this skill
  for custom integrations or debugging.
- **MCP**: MCP protocol delegates function execution back to the host; the MCP
  client implements steps 3–6 on behalf of the agent.
- **structured-tool-inputs skill**: Use before step 4 to validate and coerce
  `tool_args` into their expected types.
