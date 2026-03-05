---
name: tools-and-integrations
description: >
  Comprehensive reference for all tool types and integration patterns in Google Agent Development
  Kit (ADK) Python. Covers Function Tools (`FunctionTool`, function signatures, docstrings,
  `ToolContext`, `LongRunningFunctionTool`), MCP Tools (`McpToolset`, `StdioConnectionParams`,
  `SseServerParams`, `FastMCP`, exposing ADK tools as MCP servers), OpenAPI Tools (`OpenAPIToolset`,
  `RestApiTool`, spec loading, auto-generated tools), Tool Authentication (`AuthScheme`,
  `AuthCredential`, `token_to_scheme_credential`, OAuth2, API Key, OIDC, `ToolContext.request_credential`),
  Tool Performance (naming, descriptions, parameter design, error handling, parallel execution),
  Action Confirmations (`require_confirmation`, `request_confirmation`, `ToolConfirmation`,
  human-in-the-loop patterns), Google Search Grounding (`google_search`, `GoogleSearchTool`,
  `groundingMetadata`, agent-as-tool pattern), and Vertex AI Search Grounding (`VertexAiSearchTool`,
  enterprise datastores, RAG patterns). Consult when building any agent that needs to call external
  APIs, use MCP servers, authenticate against services, require user approval for actions, or
  ground responses in web/enterprise search results.
metadata:
  author: adk-skills
  version: "1.0"
---

# Tools & Integrations — Detailed Reference

## Overview

This skill covers every tool type and integration pattern available in Google ADK (Python). Tools are the mechanism by which agents interact with the outside world — calling APIs, reading files, searching the web, querying databases, and modifying external state. ADK provides a layered tool ecosystem:

1. **Function Tools** — custom Python functions wrapped as callable tools via `FunctionTool`
2. **MCP Tools** — tools exposed by external MCP (Model Context Protocol) servers, consumed via `McpToolset`
3. **OpenAPI Tools** — auto-generated tools from OpenAPI v3 specifications via `OpenAPIToolset`
4. **Tool Authentication** — credential management for authenticated API calls via `AuthScheme` and `AuthCredential`
5. **Tool Performance** — design patterns for reliable, high-performance tool usage
6. **Action Confirmations** — human-in-the-loop approval flows via `require_confirmation` and `request_confirmation`
7. **Google Search Grounding** — real-time web grounding via `google_search` / `GoogleSearchTool`
8. **Vertex AI Search Grounding** — enterprise document grounding via `VertexAiSearchTool`

Tools sit in the **tool layer** of the ADK architecture, invoked by the LLM during the Runner event loop. When the LLM generates a `FunctionCall`, the Runner dispatches it to the corresponding tool, executes it, packages the result as a `FunctionResponse` event (including any `state_delta` or `artifact_delta` in `EventActions`), and feeds the result back to the LLM for the next step.

For session state management within tools, see `sessions-state-memory.md`.
For callbacks that intercept tool execution, see `callbacks.md`.

---

## Prerequisites

- Python 3.10+
- `pip install google-adk`
- For MCP tools: `pip install mcp` (the `model-context-protocol` Python library); Node.js + `npx` for community MCP servers distributed as npm packages
- For OpenAPI tools: No additional dependencies beyond `google-adk`
- For Google Search Grounding: Vertex AI API enabled, `GOOGLE_GENAI_USE_VERTEXAI=TRUE`, `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_LOCATION` environment variables set
- For Vertex AI Search Grounding: Same as Google Search, plus a Vertex AI Search Data Store with its Data Store ID
- For Tool Authentication with OAuth2: OAuth client credentials (client ID/secret) configured in your GCP project or external identity provider
- For Action Confirmations: ADK v1.14.0+ for `require_confirmation`; `ResumabilityConfig` for advanced confirmation flows

---

## Core Concept

### The Tool Execution Cycle

```
User Message
    │
    ▼
┌──────────┐    FunctionCall     ┌──────────────┐
│   LLM    │ ─────────────────►  │  Tool Layer  │
│ (Gemini) │                     │              │
│          │ ◄───────────────── │  FunctionTool │
└──────────┘   FunctionResponse  │  McpToolset   │
    │                            │  OpenAPIToolset│
    ▼                            │  GoogleSearch  │
Final Response                   │  VertexSearch  │
                                 └──────────────┘
```

1. The LLM analyzes the user query and decides which tool(s) to invoke.
2. It emits a `FunctionCall` with the tool name and arguments.
3. The Runner dispatches the call to the matching tool implementation.
4. The tool executes, optionally modifying state via `ToolContext.state` or signaling actions via `ToolContext.actions`.
5. The result is wrapped in a `FunctionResponse` event and fed back to the LLM.
6. The LLM incorporates the result into its next response (or calls another tool).

### How the LLM Discovers Tools

The LLM sees each tool as a function schema consisting of a **name**, **description**, and **parameter definitions**. For `FunctionTool`, ADK auto-generates this schema from the Python function's name, docstring, type hints, and default values. For `McpToolset` and `OpenAPIToolset`, schemas are discovered from the MCP server or OpenAPI spec respectively.

The quality of tool names, descriptions, and parameter documentation directly determines how reliably the LLM selects and calls the correct tool.

---

## When to Use

- When an agent needs to call any external API, service, or system (Function Tools, OpenAPI Tools, MCP Tools).
- When you want to wrap an existing Python function as a tool without writing a schema manually (Function Tools).
- When integrating with external services that expose an MCP interface — file systems, databases, SaaS platforms (MCP Tools).
- When you have an OpenAPI v3 spec for a REST API and want auto-generated tools for every endpoint (OpenAPI Tools).
- When API calls require authentication — API keys, OAuth2, OIDC, service accounts (Tool Authentication).
- When a tool performs a destructive or costly action that requires user confirmation before execution (Action Confirmations).
- When the agent must provide factual, real-time answers grounded in web search results with citations (Google Search Grounding).
- When the agent must answer questions using private enterprise documents indexed in Vertex AI Search (Vertex AI Search Grounding).
- When you need to optimize tool design to reduce hallucinated tool calls and improve reliability (Tool Performance).

---

## Rules

### Function Tools
- The function name becomes the tool name the LLM sees. Use clear, descriptive, verb-based names (e.g., `get_weather`, `create_order`).
- The function docstring becomes the tool description. The LLM reads this to decide when to use the tool. A missing or vague docstring causes unreliable tool selection.
- All parameters must have type hints. Parameters without default values are treated as required by the LLM.
- `tool_context: ToolContext` must always be the **last** parameter if included. ADK auto-injects it; it is excluded from the schema the LLM sees.
- `*args` and `**kwargs` are ignored by the schema generator — the LLM cannot pass values to them.
- The preferred return type is `dict`. Non-dict returns are auto-wrapped as `{"result": <value>}`.
- Assigning a plain function to an agent's `tools` list auto-wraps it as `FunctionTool`. Explicit wrapping via `FunctionTool(func=my_func)` is optional but required when setting `require_confirmation`.

### MCP Tools
- `McpToolset` can be placed directly in an agent's `tools` list for use with `adk web`. For programmatic use outside `adk web`, use `MCPToolset.from_server()` which returns `(tools, exit_stack)`.
- Connection types: `StdioConnectionParams` (wrapping `StdioServerParameters` from `mcp`) for local subprocess servers, `SseServerParams` for remote HTTP SSE servers.
- MCP connections are stateful and persistent. You must explicitly close them via `exit_stack.aclose()` when done.
- `tool_filter` parameter on `McpToolset` limits which MCP tools are exposed to the agent.

### OpenAPI Tools
- `OpenAPIToolset` accepts specs as `spec_str` (JSON or YAML string) or `spec_dict` (Python dict).
- Each API operation in the spec becomes a `RestApiTool` named after the `operationId` (converted to snake_case).
- Authentication is applied at the toolset level via `auth_scheme` and `auth_credential` constructor parameters.
- The spec must be OpenAPI v3.x. Swagger 2.0 specs must be converted first.

### Tool Authentication
- `AuthScheme` defines how the API expects credentials (API Key, HTTP Bearer, OAuth2, OpenID Connect).
- `AuthCredential` holds the initial credential data (API key value, OAuth client ID/secret, service account token).
- The helper `token_to_scheme_credential(scheme_type, in_location, name, token)` generates both `AuthScheme` and `AuthCredential` for simple token-based auth.
- For OAuth2 flows requiring user consent, the framework emits an `adk_request_credential` function call that the client application must handle interactively.
- Credentials stored in session state should be encrypted when using persistent `SessionService` backends.

### Action Confirmations
- `FunctionTool(func, require_confirmation=True)` adds a boolean confirmation step — the agent pauses and asks the user to confirm before executing.
- `require_confirmation` can also be a callable that receives the same arguments as the tool function and returns `True` if confirmation is needed (conditional confirmation).
- For advanced confirmation with structured data, use `tool_context.request_confirmation(hint, payload)` inside the tool function body.
- Advanced confirmations require `ResumabilityConfig(is_resumable=True)` on the `App` object.
- `require_confirmation` is designed for single-agent workflows tightly coupled with a UI. For multi-agent or A2A workflows, use `LongRunningFunctionTool` instead.

### Grounding Tools
- `google_search` (from `google.adk.tools`) and `GoogleSearchTool` (from `google.adk.tools.google_search_tool`) cannot be mixed with other tools in the same agent (tool limitation in ADK ≤ v1.15.0; workaround available in v1.16.0+ via `bypass_multi_tools_limit=True`).
- The standard pattern is to create a dedicated search agent and use it as an `AgentTool` in a parent agent.
- `VertexAiSearchTool` requires a `data_store_id` (or `search_engine_id`) pointing to an existing Vertex AI Search Data Store.
- Both grounding tools return `groundingMetadata` in the response, containing source URLs/documents for attribution.

---

## Example

### Complete Example: Function Tool with ToolContext, State Updates, and Runner Execution

```python
import asyncio
from google.adk.agents import LlmAgent
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.adk.tools import ToolContext
from google.genai.types import Content, Part

APP_NAME = "tools_demo"
USER_ID = "user1"
MODEL = "gemini-2.0-flash"


def get_weather(city: str, tool_context: ToolContext) -> dict:
    """Retrieves the current weather for a given city.

    Args:
        city: The name of the city to get weather for.
        tool_context: Provided by ADK framework.

    Returns:
        A dictionary containing the weather information.
    """
    # Simulated weather data
    weather_data = {
        "New York": {"temp_f": 72, "condition": "Sunny"},
        "London": {"temp_f": 58, "condition": "Cloudy"},
        "Tokyo": {"temp_f": 80, "condition": "Humid"},
    }
    result = weather_data.get(city, {"temp_f": 0, "condition": "Unknown"})

    # Write to session state via ToolContext (automatically tracked)
    tool_context.state["temp:last_city_queried"] = city
    tool_context.state["user:preferred_unit"] = "fahrenheit"

    return {"city": city, **result}


weather_agent = LlmAgent(
    model=MODEL,
    name="WeatherAgent",
    instruction=(
        "You are a weather assistant. Use the get_weather tool to "
        "answer weather questions. Report temperatures in Fahrenheit."
    ),
    tools=[get_weather],  # Auto-wrapped as FunctionTool
)

session_service = InMemorySessionService()
runner = Runner(
    agent=weather_agent,
    app_name=APP_NAME,
    session_service=session_service,
)


async def main():
    session = await session_service.create_session(
        app_name=APP_NAME, user_id=USER_ID, session_id="s1"
    )

    msg = Content(parts=[Part(text="What's the weather in Tokyo?")], role="user")
    async for event in runner.run_async(
        user_id=USER_ID, session_id="s1", new_message=msg
    ):
        if event.is_final_response() and event.content and event.content.parts:
            print(f"Agent: {event.content.parts[0].text}")

    # Verify state was updated by the tool
    updated = await session_service.get_session(
        app_name=APP_NAME, user_id=USER_ID, session_id="s1"
    )
    print(f"Last city queried: {updated.state.get('temp:last_city_queried')}")


# asyncio.run(main())
```

---

## Decision Table

| Need | Recommended Tool Type | Why |
|---|---|---|
| Wrap a custom Python function as a tool | `FunctionTool` (or plain function in `tools` list) | Simplest path; auto-generates schema from signature and docstring |
| Call tools from an external MCP server | `McpToolset` with `StdioConnectionParams` or `SseServerParams` | Bridges MCP protocol to ADK; discovers tools automatically |
| Auto-generate tools from a REST API spec | `OpenAPIToolset` with OpenAPI v3 spec | Creates a `RestApiTool` for every API operation without manual coding |
| Authenticate API calls with API key | `token_to_scheme_credential("apikey", ...)` + `OpenAPIToolset` or `RestApiTool` | Simple helper for token-based auth |
| Authenticate with OAuth2 (user consent) | `AuthScheme` (OAuth2) + interactive `adk_request_credential` flow | Handles OAuth redirect and token exchange |
| Require user approval before tool execution | `FunctionTool(func, require_confirmation=True)` | Boolean or conditional confirmation with UI dialog |
| Advanced HITL with structured data collection | `tool_context.request_confirmation(hint, payload)` + `ResumabilityConfig` | Collects structured input from user before proceeding |
| HITL in multi-agent or A2A workflows | `LongRunningFunctionTool` | Async approval pattern that works across agent boundaries |
| Ground responses in real-time web results | `google_search` tool or `GoogleSearchTool` in a dedicated agent | Provides citations and up-to-date information |
| Ground responses in enterprise documents | `VertexAiSearchTool(data_store_id=...)` | Searches private Vertex AI Search datastores |
| Mix grounding tools with custom tools | Agent-as-Tool pattern (search agent as `AgentTool`) | Workaround for built-in tool isolation limitation |
| Run a long-running external operation | `LongRunningFunctionTool` | Pauses agent execution; resumes when operation completes |
| Expose ADK tools as MCP servers | `FastMCP` + `adk_to_mcp_tool_type` conversion utility | Makes ADK tools accessible to any MCP client |

---

## Common Patterns

### Pattern 1: Function Tool with Explicit FunctionTool Wrapping

When you need to set options like `require_confirmation`, explicitly wrap the function.

```python
from google.adk.agents import LlmAgent
from google.adk.tools import FunctionTool, ToolContext


def delete_record(record_id: str, tool_context: ToolContext) -> dict:
    """Permanently deletes a record by its ID.

    Args:
        record_id: The unique identifier of the record to delete.
        tool_context: Provided by ADK framework.

    Returns:
        Confirmation of the deletion.
    """
    # Simulate deletion
    return {"status": "deleted", "record_id": record_id}


agent = LlmAgent(
    model="gemini-2.0-flash",
    name="RecordManager",
    instruction="Help users manage records. Always confirm before deleting.",
    tools=[
        FunctionTool(func=delete_record, require_confirmation=True),
    ],
)
```

### Pattern 2: MCP Tools via Stdio (Local MCP Server)

Connect to a local MCP server (e.g., filesystem) using stdio transport.

```python
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StdioConnectionParams
from mcp import StdioServerParameters

root_agent = LlmAgent(
    model="gemini-2.0-flash",
    name="FileSystemAgent",
    instruction="Help users read and list files using the file system tools.",
    tools=[
        McpToolset(
            connection_params=StdioConnectionParams(
                server_params=StdioServerParameters(
                    command="npx",
                    args=["-y", "@modelcontextprotocol/server-filesystem",
                          "/path/to/allowed/folder"],
                )
            ),
            # tool_filter=["read_file", "list_directory"],  # Optional filter
        ),
    ],
)
```

### Pattern 3: MCP Tools via SSE (Remote MCP Server)

Connect to a remote MCP server using SSE transport.

```python
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, SseServerParams

# For use outside adk web, use MCPToolset.from_server()
async def create_agent():
    tools, exit_stack = await MCPToolset.from_server(
        connection_params=SseServerParams(
            url="http://localhost:8001/sse",
        )
    )
    agent = LlmAgent(
        model="gemini-2.0-flash",
        name="RemoteMCPAgent",
        instruction="Use the available tools to assist the user.",
        tools=tools,
    )
    return agent, exit_stack

# Remember to close: await exit_stack.aclose()
```

### Pattern 4: OpenAPI Tools with API Key Authentication

Auto-generate tools from an OpenAPI spec with authentication.

```python
from google.adk.agents import LlmAgent
from google.adk.tools.openapi_tool.openapi_spec_parser.openapi_toolset import (
    OpenAPIToolset,
)
from google.adk.tools.openapi_tool.auth.auth_helpers import (
    token_to_scheme_credential,
)

# Load your OpenAPI spec
with open("api_spec.json", "r") as f:
    spec_content = f.read()

# Create auth scheme and credential using the helper
auth_scheme, auth_credential = token_to_scheme_credential(
    "apikey",   # scheme type
    "header",   # in: header, query, or cookie
    "X-API-Key",  # parameter name
    "your-api-key-value",  # actual key
)

toolset = OpenAPIToolset(
    spec_str=spec_content,
    spec_str_type="json",
    auth_scheme=auth_scheme,
    auth_credential=auth_credential,
)

agent = LlmAgent(
    model="gemini-2.0-flash",
    name="APIAgent",
    instruction="Use the API tools to answer user questions.",
    tools=[toolset],
)
```

### Pattern 5: Google Search Grounding via Agent-as-Tool Pattern

Use a dedicated search agent as a tool in a parent agent to work around the built-in tool isolation limitation.

```python
from google.adk.agents import Agent
from google.adk.tools import google_search
from google.adk.tools.agent_tool import AgentTool

# Dedicated search agent (uses google_search exclusively)
search_agent = Agent(
    model="gemini-2.0-flash",
    name="SearchAgent",
    instruction="You are a search specialist. Use Google Search to answer questions.",
    tools=[google_search],
)

# Parent agent that uses search agent as a tool alongside custom tools
root_agent = Agent(
    model="gemini-2.0-flash",
    name="AssistantAgent",
    instruction=(
        "You are a helpful assistant. Use the SearchAgent tool when you need "
        "real-time information from the web. Use other tools for specific tasks."
    ),
    tools=[
        AgentTool(agent=search_agent),
        # ... other custom function tools
    ],
)
```

### Pattern 6: Vertex AI Search Grounding for Enterprise Data

Ground agent responses in private enterprise documents.

```python
from google.adk.agents import LlmAgent
from google.adk.tools import VertexAiSearchTool

# Requires: Vertex AI Search Data Store with indexed documents
# Set env vars: GOOGLE_CLOUD_PROJECT, GOOGLE_CLOUD_LOCATION
# Set .env: GOOGLE_GENAI_USE_VERTEXAI=TRUE

enterprise_agent = LlmAgent(
    model="gemini-2.0-flash",
    name="EnterpriseSearchAgent",
    instruction=(
        "Answer questions about company policies and documents. "
        "Use the Vertex AI Search tool to find relevant information."
    ),
    tools=[
        VertexAiSearchTool(
            data_store_id="your-data-store-id",
        ),
    ],
)
```

### Pattern 7: Conditional Action Confirmation

Require confirmation only when a condition is met (e.g., amount exceeds threshold).

```python
from google.adk.agents import Agent
from google.adk.tools import FunctionTool


def process_refund(order_id: str, amount: float) -> dict:
    """Processes a refund for the given order.

    Args:
        order_id: The order ID to refund.
        amount: The refund amount in dollars.

    Returns:
        Confirmation of the refund processing.
    """
    return {"status": "refunded", "order_id": order_id, "amount": amount}


def needs_confirmation(order_id: str, amount: float) -> bool:
    """Require confirmation for refunds over $100."""
    return amount > 100.0


agent = Agent(
    model="gemini-2.0-flash",
    name="RefundAgent",
    instruction="Help users process refunds. The system will ask for confirmation on large amounts.",
    tools=[
        FunctionTool(func=process_refund, require_confirmation=needs_confirmation),
    ],
)
```

---

## Edge Cases

### Built-In Tool Isolation

`GoogleSearchTool`, `VertexAiSearchTool`, and `BuiltInCodeExecutor` cannot be used alongside other tools in the same agent (ADK ≤ v1.15.0). Attempting to mix them causes errors. In ADK v1.16.0+, use `bypass_multi_tools_limit=True` to lift this restriction for `GoogleSearchTool` and `VertexAiSearchTool`. The recommended pattern remains the agent-as-tool approach: create a dedicated grounding agent and wrap it with `AgentTool`.

### MCP Connection Lifecycle and Cleanup

`McpToolset` establishes stateful, persistent connections. If the connection is not explicitly closed (via `exit_stack.aclose()`), resources leak and the MCP server process may remain running. When using `McpToolset` directly in an agent's `tools` list with `adk web`, cleanup is handled automatically. When using `MCPToolset.from_server()` programmatically, always close the `exit_stack` in a `try/finally` block.

### Missing or Vague Tool Docstrings

If a function tool has no docstring, the LLM has no description to guide its tool selection. This causes the model to either never call the tool or call it incorrectly with hallucinated arguments. Every function tool must have a clear, concise docstring describing what it does, when to use it, and what each parameter means.

### ToolContext Parameter Ordering

`tool_context: ToolContext` must be the last parameter in the function signature. If placed before other parameters, the auto-generated schema may be incorrect, and the LLM may attempt to pass a value for `tool_context` (which it should not). ADK specifically filters out the last parameter named `tool_context` from the schema.

### OpenAPIToolset with Multiple Security Schemes

`OpenAPIToolset` only supports a single security scheme per endpoint. If your OpenAPI spec defines multiple security requirements for an endpoint, only the first one is used. This is a known limitation. Workaround: create separate `RestApiTool` instances with different credentials for endpoints requiring different auth.

### Confirmation Flow Not Resuming

If `require_confirmation` is used without `ResumabilityConfig` and the advanced `request_confirmation` API, the confirmation may fail with "app is not resumable" errors. For boolean confirmations in `adk web`, this is handled automatically. For programmatic use, ensure your `App` has `ResumabilityConfig(is_resumable=True)`.

### State Key Collisions Between Tools

Multiple tools writing to the same `session.state` key causes silent overwrites. The last tool to execute wins. Namespace your state keys by tool name: `tool_name:key` (e.g., `weather:last_city`, `refund:last_amount`).

### Grounding Metadata Not Available in Events

When using `google_search` via `AgentTool`, the `groundingMetadata` (source URLs, citations) may not appear directly in the parent agent's event stream. It is available on the grounding agent's internal events. To access it, you may need to inspect the sub-agent's events or use callbacks.

### SseServerParams Import Location

`SseServerParams` is imported from `google.adk.tools.mcp_tool.mcp_toolset`, not from the `mcp` library. `StdioServerParameters` comes from `mcp`. Mixing these up causes import errors.

---

## Failure Handling

### Tool Function Raises an Exception

If a `FunctionTool`'s underlying function raises an exception, ADK catches it and returns the error message to the LLM as a `FunctionResponse` with the error content. The LLM then decides how to proceed — it may retry, try a different tool, or inform the user. The exception is not propagated to the caller of `runner.run_async()`.

### MCP Server Connection Failure

If the MCP server is unreachable (wrong path, server not running, network issues), `McpToolset` raises a connection error during tool initialization. The agent cannot start. Validate server availability before creating the agent, and handle the error at the application level.

### OpenAPI Spec Parsing Failure

If the OpenAPI spec has syntax errors, unresolvable `$ref` references, or is not v3.x, `OpenAPIToolset` raises a parsing error during initialization. Validate your spec with an external tool (e.g., Swagger Editor) before passing it to ADK.

### Authentication Token Expiry

When using `OpenAPIToolset` or `RestApiTool` with time-limited tokens (e.g., OAuth access tokens, JWTs), expired tokens cause API call failures. The tool returns the HTTP error to the LLM. Implement token refresh logic outside ADK (e.g., refresh before each session) or use the interactive `adk_request_credential` flow for re-authentication.

### Grounding Service Quota Exceeded

Google Search Grounding and Vertex AI Search are subject to Google Cloud API quotas. Exceeding quota causes the grounding tool to return an error, which the LLM reports to the user. Monitor your quota usage in the Google Cloud Console.

---

## Scalability Notes

### Tool Count and LLM Performance

The LLM receives schemas for all tools registered on an agent. As tool count increases, the LLM must process more schema tokens, reducing response quality and increasing latency. Keep tool count per agent under 20 for optimal performance. Use the agent-as-tool pattern to distribute tools across specialist agents.

### MCP Connection Statefulness

MCP connections are stateful and persistent, posing challenges for scaled deployments. Each MCP connection consumes a process or network socket. In multi-instance deployments, each instance maintains its own connections. Consider MCP server scalability and use SSE transport with load-balanced endpoints for production.

### OpenAPI Toolset Scale

Large OpenAPI specs (hundreds of endpoints) generate many `RestApiTool` instances. Use `toolset.get_tool("operation_id")` to select specific tools rather than passing the entire toolset. The LLM performs better with fewer, focused tools.

### Grounding API Quotas

Google Search Grounding and Vertex AI Search incur API calls to Google services. Both are subject to rate limits and quotas defined by your Google Cloud project. For high-throughput agents, request quota increases proactively.

### Token Budget for Grounded Responses

Grounded responses include source metadata (URLs, document chunks, citations) that consume context window tokens. Deep grounding with many sources can significantly increase token usage. Configure search result limits where possible.

---

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
|---|---|---|
| Function tool with no docstring | LLM cannot determine when or how to use the tool | Always write a clear, specific docstring describing purpose, parameters, and return value |
| Putting `tool_context` before other parameters | Schema generator may include `tool_context` in the LLM-visible schema | Always make `tool_context: ToolContext` the **last** parameter |
| Returning raw error codes from tools | LLM cannot interpret numeric codes | Return descriptive error dicts: `{"error": "City not found", "suggestion": "Try 'New York'"}` |
| Mixing `GoogleSearchTool` with custom tools in one agent | Built-in tool isolation causes runtime errors | Use agent-as-tool pattern: dedicated search agent wrapped as `AgentTool` |
| Not closing MCP `exit_stack` | Resource leak; MCP server process remains running | Always close with `await exit_stack.aclose()` in a `finally` block |
| Passing entire large OpenAPI spec without filtering | LLM overwhelmed by hundreds of tool schemas | Select specific tools via `toolset.get_tool()` or use multiple specialized agents |
| Storing OAuth tokens in plain-text session state with persistent `SessionService` | Security risk — tokens readable from database | Encrypt tokens before storage; use secure credential management |
| Using `require_confirmation` in multi-agent/A2A systems | Confirmation does not propagate across agent boundaries | Use `LongRunningFunctionTool` for async approval patterns |
| Overly generic tool names like `do_task` or `handle_request` | LLM cannot distinguish between tools, leading to wrong calls | Use specific verb-noun names: `get_weather`, `create_ticket`, `delete_record` |
| Single monolithic tool doing many things | LLM struggles to select correct parameters for multi-purpose tools | Split into multiple focused tools with single responsibilities |

---

## Guidelines

1. **Write precise docstrings for every `FunctionTool`.** The docstring is the tool's description that the LLM reads to decide when and how to call it. Include what the tool does, when it should be used, and what each parameter represents. Vague or missing docstrings are the #1 cause of unreliable tool usage.

2. **Use the agent-as-tool pattern for built-in grounding tools.** Create dedicated agents for `google_search` or `VertexAiSearchTool`, then wrap them with `AgentTool` for use in parent agents that also have custom function tools. This avoids the built-in tool isolation limitation and keeps tool responsibilities clean.

3. **Always close MCP connections explicitly.** When using `MCPToolset.from_server()` outside of `adk web`, store the `exit_stack` and call `await exit_stack.aclose()` in a `finally` block to prevent resource leaks and orphaned server processes.

4. **Use `token_to_scheme_credential` for simple API key and bearer token authentication.** This helper from `google.adk.tools.openapi_tool.auth.auth_helpers` generates both `AuthScheme` and `AuthCredential` in one call, reducing boilerplate for `OpenAPIToolset` configuration.

5. **Return structured dictionaries from function tools.** Include descriptive keys like `status`, `result`, `error_message`, and contextual data the LLM can use to form a natural response. Avoid returning raw strings, numeric codes, or large unstructured payloads.

6. **Use conditional confirmation for risk-proportionate approval.** Pass a callable to `require_confirmation` that evaluates the tool's arguments and returns `True` only when confirmation is warranted (e.g., high-value transactions, destructive operations). This avoids confirmation fatigue for low-risk actions.

7. **Limit tools per agent to ~20 for reliable LLM tool selection.** If you have more tools, distribute them across specialist agents and use `AgentTool` or `sub_agents` to route requests to the appropriate specialist.

8. **Validate OpenAPI specs externally before loading into `OpenAPIToolset`.** Use Swagger Editor or similar tools to verify the spec is valid OpenAPI v3.x with correct `$ref` resolution. Invalid specs cause initialization-time errors that are hard to debug within ADK.

---

## References

- [Official ADK Docs — Custom Tools Overview](https://google.github.io/adk-docs/tools-custom/)
- [Official ADK Docs — Function Tools](https://google.github.io/adk-docs/tools-custom/function-tools/)
- [Official ADK Docs — MCP Tools](https://google.github.io/adk-docs/tools-custom/mcp-tools/)
- [Official ADK Docs — OpenAPI Tools](https://google.github.io/adk-docs/tools-custom/openapi-tools/)
- [Official ADK Docs — Tool Authentication](https://google.github.io/adk-docs/tools-custom/authentication/)
- [Official ADK Docs — Tool Performance](https://google.github.io/adk-docs/tools-custom/performance/)
- [Official ADK Docs — Action Confirmations](https://google.github.io/adk-docs/tools-custom/confirmation/)
- [Official ADK Docs — Tool Limitations](https://google.github.io/adk-docs/tools/limitations/)
- [Official ADK Docs — Google Search Grounding](https://google.github.io/adk-docs/grounding/google_search_grounding/)
- [Official ADK Docs — Vertex AI Search Grounding](https://google.github.io/adk-docs/grounding/vertex_ai_search_grounding/)
- [Official ADK Docs — MCP Integration Overview](https://google.github.io/adk-docs/mcp/)
- [ADK Python API Reference](https://google.github.io/adk-docs/api-reference/python/)
- [ADK Python GitHub](https://github.com/google/adk-python)
