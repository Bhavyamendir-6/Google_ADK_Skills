---
name: tool-registration
description: >
  Use this skill when you need to register one or more defined tools with an
  LLM agent framework (Google ADK, OpenAI Agents SDK, MCP, LangChain, etc.)
  so the agent can discover and invoke them during a run.
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

This skill covers the mechanics of attaching already-defined tool functions to
an agent runtime so that the LLM can discover, select, and invoke them. Tool
registration differs from tool definition: definition is the function contract;
registration is the act of binding it to an agent's tool list.

## When to Use

- When you have a defined tool function and need to make it available to an agent.
- When adding, removing, or replacing tools in an existing agent configuration.
- When migrating tools from one agent framework to another.

## When NOT to Use

- Do not use this skill to write the tool function itself (see `tool-definition`).
- Do not use for MCP server setup—that is a server configuration concern.
- Do not use when the tool is already registered and the issue is in calling logic
  (see `function-calling`).

## Capabilities

- Registers Python functions as tools on Google ADK `LlmAgent`.
- Registers tools on OpenAI `Agent` via the `tools` parameter.
- Registers tools on LangChain agents via `AgentExecutor`.
- Produces the minimal boilerplate for each framework.
- Handles registration of multiple tools at once.

## Step-by-Step Instructions

1. **Verify the tool is already defined** with correct type annotations and
   docstring. If not, complete `tool-definition` first.

2. **Identify the target framework** from context:
   - Google ADK → `LlmAgent(tools=[...])`
   - OpenAI Agents SDK → `Agent(tools=[...])`
   - LangChain → `create_tool_calling_agent` + `AgentExecutor`
   - MCP → `server.tool()` decorator

3. **Import the tool function** at the top of the agent module.

4. **Pass the function directly** into the agent's `tools` list. Do NOT wrap
   it in any extra class unless the framework specifically requires it.

5. **For Google ADK**: Wrap the function with `FunctionTool` only if you need
   to override metadata. Otherwise pass the raw function.

6. **For MCP servers**: Use the `@server.tool()` decorator on the function
   definition itself—registration and definition are combined.

7. **Confirm the tool appears in the agent's tool list** by printing or logging
   `agent.tools` before the first run.

8. **Run a smoke test** with a prompt that explicitly requires the tool to
   verify end-to-end registration.

## Input Format

```
framework: google-adk | openai-agents | langchain | mcp
tools:
  - <function_name_1>
  - <function_name_2>
agent_name: <str>
model: <model_identifier>
```

## Output Format

A complete agent instantiation block with tools registered:

**Google ADK example output:**
```python
from google.adk.agents import LlmAgent
from tools.weather import get_weather
from tools.search import search_web

weather_agent = LlmAgent(
    name="weather_agent",
    model="gemini-2.0-flash",
    tools=[get_weather, search_web],
    instruction="You are a helpful assistant. Use tools to answer questions.",
)
```

**OpenAI Agents SDK example output:**
```python
from agents import Agent
from tools.weather import get_weather

weather_agent = Agent(
    name="weather_agent",
    model="gpt-4o",
    tools=[get_weather],
    instructions="You are a helpful assistant.",
)
```

## Error Handling

- **Tool not found at import time**: Verify module path and `__init__.py` exports.
- **Schema extraction fails**: The tool function is missing type annotations or
  docstring. Apply `tool-definition` skill first.
- **Duplicate tool names**: Each registered tool must have a unique `__name__`.
  Rename conflicting functions before registration.
- **Framework version mismatch**: If `tools` parameter is not accepted, check
  the framework version and consult its changelog.

## Examples

### Example 1 — Google ADK multi-tool agent

```python
from google.adk.agents import LlmAgent
from my_tools import get_stock_price, send_slack_message, search_products

agent = LlmAgent(
    name="assistant",
    model="gemini-2.0-flash",
    tools=[get_stock_price, send_slack_message, search_products],
    instruction="Use the available tools to answer user queries.",
)
```

### Example 2 — OpenAI Agents SDK

```python
from agents import Agent
from my_tools import get_stock_price, send_slack_message

agent = Agent(
    name="finance_bot",
    model="gpt-4o",
    tools=[get_stock_price, send_slack_message],
    instructions="Answer financial queries using the tools provided.",
)
```

### Example 3 — MCP server registration

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-tools-server")

@mcp.tool()
def get_stock_price(ticker: str, currency: str = "USD") -> dict:
    """Retrieves the current stock price for a given ticker symbol.

    Args:
        ticker (str): The stock ticker symbol.
        currency (str): The currency. Defaults to 'USD'.

    Returns:
        dict: Contains 'ticker', 'price', and 'currency'.
    """
    ...
```

## Edge Cases

- **Dynamic tool registration**: If tools must be added at runtime, store them
  in a mutable list and reassign `agent.tools` before the next run if the
  framework supports it.
- **Tool with same name as built-in**: Prefix the function name with a domain
  prefix (e.g., `finance_search` not `search`) to avoid collisions.
- **Disabling a tool temporarily**: Remove it from the `tools` list rather than
  commenting out the definition—the definition must remain importable.

## Integration Notes

- **Google ADK**: `LlmAgent` accepts raw Python callables; ADK extracts the
  schema from type hints and docstrings automatically.
- **OpenAI Agents SDK**: Wraps functions using `function_tool()` internally;
  passing raw functions is supported as of SDK v1+.
- **MCP**: The `@server.tool()` decorator simultaneously defines and registers
  the tool with the MCP server's tool registry.
- **tool-chaining skill**: Registered tools are prerequisites for the
  `tool-chaining` skill, which governs sequential multi-tool invocation.
