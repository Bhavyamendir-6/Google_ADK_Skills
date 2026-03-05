---
name: callbacks-and-control-flow
description: >
  Comprehensive reference for the callback system and callback design patterns in Google Agent
  Development Kit (ADK) Python. Covers all six callback hooks (`before_agent_callback`,
  `after_agent_callback`, `before_model_callback`, `after_model_callback`, `before_tool_callback`,
  `after_tool_callback`), callback signatures and return types (`CallbackContext`, `ToolContext`,
  `LlmRequest`, `LlmResponse`, `types.Content`), the return-value control flow mechanism
  (`None` to proceed, specific object to override/skip), callback registration on `LlmAgent`
  and `BaseAgent`, callback list support (multiple callbacks per hook with short-circuit
  ordering), execution flow lifecycle (user request → before_agent → before_model → LLM →
  after_model → before_tool → tool → after_tool → after_agent → response), and production
  design patterns including guardrails & policy enforcement, input/output validation, dynamic
  state management, logging & monitoring, caching (before/after paired pattern), request/response
  modification, conditional step skipping, tool-specific actions (`request_credential`,
  `skip_summarization`), artifact handling in callbacks, content moderation, and rate limiting.
  Also covers the relationship between agent-level callbacks and `BasePlugin` runner-level
  callbacks (plugin callbacks take precedence), error handling in callbacks, state prefix
  conventions (`app:`, `user:`, `temp:`), and idempotency considerations. Consult when
  implementing guardrails, logging, caching, content filtering, authentication flows, or
  any custom behavior at agent execution checkpoints.
metadata:
  author: adk-skills
  version: "1.0"
---

# Callbacks & Control Flow — Detailed Reference

## Overview

This skill covers the complete callback system in Google ADK (Python) — six hooks that let you observe, customize, and control agent behavior at every critical execution point. Callbacks are the primary mechanism for guardrails, logging, caching, and production-ready agent behavior.

1. **Callback System Fundamentals** — the six hooks, their signatures, context objects, and the return-value control flow mechanism.
2. **Agent Lifecycle Callbacks** — `before_agent_callback` and `after_agent_callback` on any `BaseAgent` subclass.
3. **Model Interaction Callbacks** — `before_model_callback` and `after_model_callback` for LLM request/response interception.
4. **Tool Execution Callbacks** — `before_tool_callback` and `after_tool_callback` for tool argument validation and result processing.
5. **Callback Registration & Lists** — single function or list of callbacks per hook, short-circuit ordering.
6. **Design Patterns** — guardrails, caching, logging, state management, content moderation, rate limiting, authentication.
7. **Plugins vs Callbacks** — `BasePlugin` runner-level callbacks that apply globally and take precedence over agent callbacks.

Callbacks are available on all agent types. Agent lifecycle callbacks (`before_agent_callback`, `after_agent_callback`) are defined on `BaseAgent` and thus available on `LlmAgent`, `SequentialAgent`, `ParallelAgent`, `LoopAgent`, and custom agents. Model and tool callbacks (`before_model_callback`, `after_model_callback`, `before_tool_callback`, `after_tool_callback`) are specific to `LlmAgent`.

For context objects and state management, see `context-and-events.md`.
For session state prefixes and persistence, see `sessions-state-memory.md`.
For agent types and configuration, see `agent-architecture-types.md`.

---

## Prerequisites

- Python 3.10+
- `pip install google-adk`
- A Gemini model string (e.g., `"gemini-2.0-flash"`) or a `LiteLlm`-wrapped model
- Key imports for callbacks:
  - `from google.adk.agents import LlmAgent` (or `Agent`)
  - `from google.adk.agents.callback_context import CallbackContext`
  - `from google.adk.tools import ToolContext`
  - `from google.adk.models import LlmResponse, LlmRequest`
  - `from google.genai import types`
  - `from typing import Optional, Dict, Any`
- For Plugins: `from google.adk.plugins.base_plugin import BasePlugin`
- For running agents: `from google.adk.runners import InMemoryRunner` or `Runner`

---

## Core Concept

### Execution Flow and Hook Points

ADK provides six callback hooks that fire at specific stages of an agent's execution lifecycle:

```
User Request
    │
    ▼
┌─────────────────────┐
│  before_agent        │ ── Return Content → skip entire agent run
│  (CallbackContext)   │ ── Return None   → proceed
└─────────┬───────────┘
          ▼
    ┌─────────────────────┐
    │  before_model        │ ── Return LlmResponse → skip LLM call
    │  (CallbackContext,   │ ── Return None        → proceed
    │   LlmRequest)        │
    └─────────┬───────────┘
              ▼
        [ LLM Call ]
              │
              ▼
    ┌─────────────────────┐
    │  after_model         │ ── Return LlmResponse → replace LLM result
    │  (CallbackContext,   │ ── Return None        → use original
    │   LlmResponse)       │
    └─────────┬───────────┘
              ▼
        Tool needed? ──No──┐
              │Yes         │
              ▼            │
    ┌─────────────────────┐│
    │  before_tool         ││
    │  (BaseTool, dict,    ││ ── Return dict → skip tool, use as result
    │   ToolContext)        ││ ── Return None → proceed
    └─────────┬───────────┘│
              ▼            │
        [ Tool Runs ]      │
              │             │
              ▼            │
    ┌─────────────────────┐│
    │  after_tool          ││
    │  (BaseTool, dict,    ││ ── Return dict → replace tool result
    │   ToolContext, dict)  ││ ── Return None → use original
    └─────────┬───────────┘│
              │             │
              ▼             │
    (loop back to           │
     before_model if        │
     LLM needs another      │
     turn)                  │
              │◄────────────┘
              ▼
┌─────────────────────┐
│  after_agent         │ ── Return Content → replace agent output
│  (CallbackContext)   │ ── Return None   → use original
└─────────┬───────────┘
          ▼
      Response
```

### The Return-Value Control Mechanism

The core mechanism of ADK callbacks is the return value:

**`return None`** — observe only, proceed with default behavior. For `before_*` callbacks, the next step runs normally. For `after_*` callbacks, the result from the preceding step is used as-is.

**`return <specific object>`** — override default behavior. The framework uses the returned object and skips or replaces the normal step:

| Callback | Return to Override | Effect |
|---|---|---|
| `before_agent_callback` | `types.Content` | Skips agent's entire `_run_async_impl`; returned Content becomes agent output |
| `after_agent_callback` | `types.Content` | Replaces the Content the agent produced |
| `before_model_callback` | `LlmResponse` | Skips LLM call entirely; returned LlmResponse is processed as if from LLM |
| `after_model_callback` | `LlmResponse` | Replaces the LlmResponse received from the LLM |
| `before_tool_callback` | `dict` | Skips tool execution; returned dict is used as tool result |
| `after_tool_callback` | `dict` | Replaces the dict result the tool returned |

### Callback Signatures (Python)

```python
# Agent lifecycle — available on BaseAgent (all agent types)
def before_agent_callback(callback_context: CallbackContext) -> Optional[types.Content]:
    ...

def after_agent_callback(callback_context: CallbackContext) -> Optional[types.Content]:
    ...

# Model interaction — LlmAgent only
def before_model_callback(
    callback_context: CallbackContext, llm_request: LlmRequest
) -> Optional[LlmResponse]:
    ...

def after_model_callback(
    callback_context: CallbackContext, llm_response: LlmResponse
) -> Optional[LlmResponse]:
    ...

# Tool execution — LlmAgent only
def before_tool_callback(
    tool: BaseTool, args: Dict[str, Any], tool_context: ToolContext
) -> Optional[Dict]:
    ...

def after_tool_callback(
    tool: BaseTool, args: Dict[str, Any], tool_context: ToolContext,
    tool_response: Dict
) -> Optional[Dict]:
    ...
```

### Context Objects

**`CallbackContext`** — passed to agent and model callbacks. Provides mutable `state` property (read/write session state, changes tracked in `Event.actions.state_delta`), `agent_name`, `invocation_id`, `load_artifact()`, `save_artifact()`, and `user_content`.

**`ToolContext`** — extends `CallbackContext`. Passed to tool callbacks and tool functions. Adds `request_credential(auth_config)`, `get_auth_response(auth_config)`, `list_artifacts()`, `search_memory(query)`, `function_call_id`, and `actions` property (access to `EventActions` for `skip_summarization`, `transfer_to_agent`, `escalate`).

### Callback List Support

Each callback hook accepts either a single function or a list of functions. When a list is provided, callbacks execute in order until one returns a non-`None` value, at which point the remaining callbacks in the list are skipped (short-circuit behavior):

```python
agent = LlmAgent(
    name="guarded_agent",
    model="gemini-2.0-flash",
    instruction="Be helpful.",
    before_model_callback=[log_request, check_guardrails, inject_context],
    # Executes log_request first, then check_guardrails.
    # If check_guardrails returns LlmResponse, inject_context is skipped.
)
```

---

## When to Use

Use callbacks when you need to intercept agent execution without modifying core ADK framework code. Specific scenarios: enforce input/output guardrails before LLM calls, log every model and tool invocation for observability, cache LLM or tool results to reduce costs and latency, modify prompts dynamically based on session state, validate or sanitize tool arguments before execution, filter or redact sensitive content from LLM responses, trigger authentication flows before secure tool calls, save artifacts at specific lifecycle points, implement rate limiting or quota enforcement, or conditionally skip agent/model/tool execution.

Use `BasePlugin` instead of agent callbacks when you need the same callback logic applied globally across all agents in a runner, or when you need additional hooks like `on_user_message_callback`, `before_run_callback`, `after_run_callback`, `on_event_callback`, `on_model_error_callback`, or `on_tool_error_callback`.

---

## Rules

1. **Return `None` to proceed, return a specific object to override.** This is the fundamental control mechanism. There is no other way to control flow from within a callback.

2. **Agent callbacks are on `BaseAgent`.** `before_agent_callback` and `after_agent_callback` are available on every agent type (`LlmAgent`, `SequentialAgent`, `ParallelAgent`, `LoopAgent`, custom `BaseAgent` subclasses).

3. **Model and tool callbacks are on `LlmAgent` only.** `before_model_callback`, `after_model_callback`, `before_tool_callback`, `after_tool_callback` are parameters of `LlmAgent` (and its `Agent` alias). They are not available on workflow agents.

4. **Context type matters.** Agent and model callbacks receive `CallbackContext`. Tool callbacks receive `ToolContext` (which extends `CallbackContext` with auth, memory, and actions methods). Always use the correct type annotation.

5. **State changes in callbacks are tracked.** Modifications to `callback_context.state['key'] = value` or `tool_context.state['key'] = value` are automatically recorded in the `Event.actions.state_delta` of the event generated after the callback runs.

6. **Callback lists short-circuit.** When a list of callbacks is provided, they execute in order. The first callback that returns non-`None` stops the chain — remaining callbacks in the list are skipped.

7. **`before_agent_callback` returning `Content` must use `role="model"`.** The returned `types.Content` object must have `role="model"` to be treated as the agent's response.

8. **Sub-agents need their own callbacks.** In multi-agent systems, callbacks registered on a parent agent do not propagate to sub-agents. Each agent in the hierarchy must have its own callbacks explicitly set.

9. **Plugin callbacks take precedence over agent callbacks.** Plugins registered on the `Runner` execute before agent-level callbacks at each hook point. If a plugin callback returns a non-`None` value, the corresponding agent callback is skipped entirely.

10. **`after_agent_callback` requires consuming all events.** If you `break` out of the `runner.run_async()` event loop after `is_final_response()`, the `after_agent_callback` may not fire. The `adk run` and `adk web` commands handle this correctly, but custom runner loops must consume all events.

11. **Callbacks can be sync or async.** Both synchronous functions and `async` coroutines are accepted as callbacks. The framework awaits async callbacks automatically.

12. **Tool callback `args` dict is mutable.** In `before_tool_callback`, you can modify the `args` dictionary in-place to change tool arguments before execution. Returning `None` after modification allows the tool to proceed with the modified args.

---

## Example

A complete working example demonstrating all six callback hooks with guardrails, logging, caching, and content filtering:

```python
import logging
from typing import Optional, Dict, Any

from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.adk.models import LlmResponse, LlmRequest
from google.adk.tools import ToolContext, BaseTool, FunctionTool
from google.adk.runners import InMemoryRunner
from google.genai import types

logger = logging.getLogger(__name__)

# ── Blocked terms for guardrails ──
BLOCKED_TERMS = ["password", "ssn", "credit_card"]

# ── 1. before_agent_callback: Auth check ──
def check_auth(callback_context: CallbackContext) -> Optional[types.Content]:
    user_role = callback_context.state.get("user_role", "anonymous")
    logger.info(f"[before_agent] Agent={callback_context.agent_name}, role={user_role}")
    if user_role == "banned":
        return types.Content(
            parts=[types.Part(text="Access denied. Your account is suspended.")],
            role="model",
        )
    callback_context.state["agent_invocations"] = (
        callback_context.state.get("agent_invocations", 0) + 1
    )
    return None  # proceed

# ── 2. after_agent_callback: Append metadata ──
def add_footer(callback_context: CallbackContext) -> Optional[types.Content]:
    inv_count = callback_context.state.get("agent_invocations", 0)
    logger.info(f"[after_agent] Completed invocation #{inv_count}")
    return None  # use original output

# ── 3. before_model_callback: Input guardrail + cache check ──
def guardrail_and_cache(
    callback_context: CallbackContext, llm_request: LlmRequest
) -> Optional[LlmResponse]:
    # Extract last user message text
    last_text = ""
    if llm_request.contents:
        last_content = llm_request.contents[-1]
        if last_content.parts:
            last_text = last_content.parts[0].text or ""

    # Guardrail: block forbidden terms
    for term in BLOCKED_TERMS:
        if term.lower() in last_text.lower():
            logger.warning(f"[before_model] BLOCKED: found '{term}'")
            return LlmResponse(
                content=types.Content(
                    parts=[types.Part(text="I cannot process requests containing sensitive data.")],
                    role="model",
                )
            )

    # Cache check
    cache_key = f"temp:cache:{hash(last_text)}"
    cached = callback_context.state.get(cache_key)
    if cached:
        logger.info("[before_model] Cache HIT — returning cached response")
        return LlmResponse(
            content=types.Content(parts=[types.Part(text=cached)], role="model")
        )

    logger.info("[before_model] Proceeding to LLM")
    return None

# ── 4. after_model_callback: Cache store + output filtering ──
def cache_and_filter(
    callback_context: CallbackContext, llm_response: LlmResponse
) -> Optional[LlmResponse]:
    if llm_response.content and llm_response.content.parts:
        text = llm_response.content.parts[0].text or ""
        # Store in cache
        last_contents = callback_context.state.get("temp:last_query", "")
        if last_contents:
            cache_key = f"temp:cache:{hash(last_contents)}"
            callback_context.state[cache_key] = text

        # Output filter: redact emails
        import re
        redacted = re.sub(r'\b[\w.-]+@[\w.-]+\.\w+\b', '[REDACTED_EMAIL]', text)
        if redacted != text:
            logger.info("[after_model] Redacted email addresses from response")
            return LlmResponse(
                content=types.Content(parts=[types.Part(text=redacted)], role="model")
            )
    return None  # use original

# ── 5. before_tool_callback: Argument validation ──
def validate_tool_args(
    tool: BaseTool, args: Dict[str, Any], tool_context: ToolContext
) -> Optional[Dict]:
    logger.info(f"[before_tool] Tool={tool.name}, Args={args}")

    # Rate limiting via state
    call_count = tool_context.state.get("temp:tool_calls", 0)
    if call_count >= 10:
        logger.warning("[before_tool] Rate limit exceeded")
        return {"error": "Rate limit exceeded. Try again later."}
    tool_context.state["temp:tool_calls"] = call_count + 1

    # Argument sanitization for search tools
    if tool.name == "search_docs" and "query" in args:
        args["query"] = args["query"].strip()[:500]  # Truncate long queries

    return None  # proceed with (possibly modified) args

# ── 6. after_tool_callback: Result logging + enrichment ──
def enrich_tool_result(
    tool: BaseTool, args: Dict[str, Any], tool_context: ToolContext,
    tool_response: Dict
) -> Optional[Dict]:
    logger.info(f"[after_tool] Tool={tool.name}, Response keys={list(tool_response.keys())}")

    # Save transaction IDs to state for later reference
    if "transaction_id" in tool_response:
        tool_context.state["last_transaction_id"] = tool_response["transaction_id"]

    return None  # use original result

# ── Tool definition ──
def search_docs(query: str, tool_context: ToolContext) -> Dict[str, Any]:
    """Search internal documentation.

    Args:
        query: The search query string.
    """
    return {"status": "success", "results": [f"Doc about: {query}"], "count": 1}

# ── Agent assembly ──
agent = LlmAgent(
    name="ProductionAgent",
    model="gemini-2.0-flash",
    instruction="You are a helpful assistant. Use search_docs to find information.",
    tools=[FunctionTool(search_docs)],
    before_agent_callback=check_auth,
    after_agent_callback=add_footer,
    before_model_callback=guardrail_and_cache,
    after_model_callback=cache_and_filter,
    before_tool_callback=validate_tool_args,
    after_tool_callback=enrich_tool_result,
)

# ── Run ──
async def main():
    runner = InMemoryRunner(agent=agent, app_name="callback_demo")
    session_service = runner.session_service
    session_service.create_session(
        app_name="callback_demo",
        user_id="user1",
        session_id="session1",
        state={"user_role": "admin"},
    )
    async for event in runner.run_async(
        user_id="user1",
        session_id="session1",
        new_message=types.Content(
            role="user", parts=[types.Part(text="Search for ADK callback docs")]
        ),
    ):
        if event.is_final_response() and event.content and event.content.parts:
            print(f"Response: {event.content.parts[0].text}")

# import asyncio; asyncio.run(main())
```

---

## Decision Table

| Scenario | Callback to Use | Return Value |
|---|---|---|
| Block unauthenticated users | `before_agent_callback` | `types.Content` with denial message |
| Log every LLM request | `before_model_callback` | `None` (observe only) |
| Block prohibited input topics | `before_model_callback` | `LlmResponse` with rejection |
| Serve cached LLM responses | `before_model_callback` | `LlmResponse` with cached content |
| Store LLM response in cache | `after_model_callback` | `None` (observe + write state) |
| Redact PII from LLM output | `after_model_callback` | `LlmResponse` with filtered text |
| Validate tool arguments | `before_tool_callback` | `None` (modify args in-place) or `dict` to block |
| Rate-limit tool calls | `before_tool_callback` | `dict` with error when over limit |
| Trigger auth flow for tool | `before_tool_callback` | Call `tool_context.request_credential()`, return `None` |
| Save tool output as artifact | `after_tool_callback` | `None` (observe + save artifact) |
| Skip LLM summarization of tool | `after_tool_callback` | Set `tool_context.actions.skip_summarization = True`, return `None` |
| Apply cross-agent logging | `BasePlugin` | Register on `Runner`, implement desired hooks |

---

## Common Patterns

### 1. Guardrail (Input Blocking)

Use `before_model_callback` to inspect `llm_request.contents` for policy violations. Return an `LlmResponse` to block the LLM call entirely:

```python
def input_guardrail(
    callback_context: CallbackContext, llm_request: LlmRequest
) -> Optional[LlmResponse]:
    last_text = ""
    if llm_request.contents:
        for part in llm_request.contents[-1].parts:
            last_text += part.text or ""
    if any(word in last_text.lower() for word in FORBIDDEN_WORDS):
        callback_context.state["temp:violations"] = (
            callback_context.state.get("temp:violations", 0) + 1
        )
        return LlmResponse(
            content=types.Content(
                parts=[types.Part(text="This request violates content policy.")],
                role="model",
            )
        )
    return None
```

### 2. Paired Caching (Before + After)

Use `before_model_callback` to check cache, `after_model_callback` to populate it. This avoids redundant LLM calls:

```python
def check_cache(callback_context: CallbackContext, llm_request: LlmRequest) -> Optional[LlmResponse]:
    key = f"temp:cache:{hash(str(llm_request.contents))}"
    cached = callback_context.state.get(key)
    if cached:
        return LlmResponse(content=types.Content(parts=[types.Part(text=cached)], role="model"))
    callback_context.state["temp:pending_cache_key"] = key
    return None

def store_cache(callback_context: CallbackContext, llm_response: LlmResponse) -> Optional[LlmResponse]:
    key = callback_context.state.get("temp:pending_cache_key")
    if key and llm_response.content and llm_response.content.parts:
        callback_context.state[key] = llm_response.content.parts[0].text
    return None
```

### 3. Structured Logging

Use multiple callbacks to create an audit trail. Return `None` from all (observe-only):

```python
def log_before_model(callback_context: CallbackContext, llm_request: LlmRequest) -> None:
    logger.info(f"[{callback_context.invocation_id}] LLM request for {callback_context.agent_name}")
    return None

def log_after_tool(tool: BaseTool, args: Dict, tool_context: ToolContext, tool_response: Dict) -> None:
    logger.info(f"[{tool_context.invocation_id}] Tool {tool.name} returned {list(tool_response.keys())}")
    return None
```

### 4. Dynamic Prompt Injection

Modify `llm_request` in-place within `before_model_callback` to append context from state:

```python
def inject_user_context(
    callback_context: CallbackContext, llm_request: LlmRequest
) -> Optional[LlmResponse]:
    user_tier = callback_context.state.get("user_tier", "free")
    lang = callback_context.state.get("user_language", "en")
    if llm_request.config and llm_request.config.system_instruction:
        current = llm_request.config.system_instruction
        if isinstance(current, str):
            llm_request.config.system_instruction = (
                f"{current}\nUser tier: {user_tier}. Respond in: {lang}."
            )
    return None  # proceed with modified request
```

### 5. Tool Authentication Gate

Use `before_tool_callback` with `ToolContext` to enforce authentication before secure tools:

```python
def require_auth(tool: BaseTool, args: Dict, tool_context: ToolContext) -> Optional[Dict]:
    if tool.name in ["update_database", "send_email"]:
        auth_response = tool_context.get_auth_response(my_auth_config)
        if not auth_response:
            tool_context.request_credential(my_auth_config)
            return {"status": "auth_required", "message": "Please authenticate first."}
    return None
```

---

## Edge Cases

### 1. `after_agent_callback` Not Firing with `break`

If your custom runner loop calls `break` after `event.is_final_response()`, the `after_agent_callback` event is never consumed from the async generator. This callback fires correctly with `adk run` and `adk web` but not with premature loop termination. Solution: consume all events from `runner.run_async()` without breaking early, or accept that `after_agent_callback` will not fire.

### 2. `before_agent_callback` Content Must Have `role="model"`

When returning `types.Content` from `before_agent_callback` to skip agent execution, you must set `role="model"`. Omitting the role or using `role="user"` causes the framework to misinterpret the response.

### 3. Callback State Writes Are Tied to the Event

State changes in callbacks (`callback_context.state['key'] = value`) are tracked in the `state_delta` of the event generated *after* that callback. If a `before_model_callback` writes state and then returns an `LlmResponse` (skipping the LLM), the state change is still persisted — it is associated with the override event, not the LLM call event.

### 4. Tool Callbacks Not Firing for Certain Built-in Tools

Some built-in tools (e.g., `VertexAiSearchTool`, Google Search grounding) may not trigger `before_tool_callback` and `after_tool_callback` because they execute at the model API level rather than through ADK's tool execution pipeline. This is a known limitation. Only `FunctionTool`, `AgentTool`, `MCPTool`, and `OpenAPITool` reliably trigger tool callbacks.

### 5. Callback List Short-Circuit Skips Remaining Callbacks

When using a list of callbacks, if an early callback returns non-`None`, later callbacks never execute. A logging callback placed after a guardrail callback in the list will not log blocked requests. Place logging callbacks before guardrail callbacks in the list to ensure they always run.

### 6. Plugin Callbacks Override Agent Callbacks

If a `BasePlugin` registered on the `Runner` has a `before_model_callback` that returns an `LlmResponse`, the agent-level `before_model_callback` is never called. Plugins always execute first and take precedence. Design accordingly when combining plugins and agent callbacks.

### 7. Async vs Sync Callbacks

Both sync and async callback functions are accepted. However, if your callback performs I/O (database lookups, HTTP calls), use an `async` function to avoid blocking the event loop. The framework awaits async callbacks automatically.

### 8. Mutable `args` in `before_tool_callback`

The `args` dict passed to `before_tool_callback` is mutable. Modifying it in-place (e.g., `args["query"] = sanitized_query`) changes the arguments the tool receives, even when returning `None`. This is intentional and useful for sanitization, but accidental mutation can cause hard-to-debug issues.

### 9. `before_model_callback` Fires Multiple Times Per Request

In a single user request, the LLM may be called multiple times (initial response, then again after tool results are returned for summarization). `before_model_callback` fires before *each* LLM call, not once per user message. Design caching and guardrail logic to handle repeated invocations.

### 10. Multi-Agent Callback Isolation

Callbacks are per-agent. A `before_model_callback` on a parent agent does not intercept LLM calls made by sub-agents. Each sub-agent requires its own callbacks. In a hub-and-spoke architecture, you must register guardrails on every specialist agent individually.

---

## Failure Handling

| Failure | Symptom | Resolution |
|---|---|---|
| Callback raises unhandled exception | Agent invocation may crash or return error event | Wrap all callback logic in `try/except`; log errors; return `None` to allow default behavior on failure |
| `before_model_callback` returns wrong type (e.g., `dict`) | TypeError or unexpected behavior | Must return `Optional[LlmResponse]`, not `dict` or `Content` |
| `before_tool_callback` returns `LlmResponse` | TypeError or ignored return | Must return `Optional[dict]`, not `LlmResponse` or `Content` |
| `before_agent_callback` returns `LlmResponse` | Framework ignores override | Must return `Optional[types.Content]` with `role="model"` |
| State key collision across callbacks | Unexpected values overwritten | Use namespaced keys with `temp:`, `app:`, or `user:` prefixes |
| Callback modifies `llm_request` after returning `LlmResponse` | Modifications ignored since LLM call is skipped | Only modify `llm_request` in-place when returning `None` |
| `after_agent_callback` never fires | Using `break` in `run_async()` loop | Consume all events; do not break early from the async generator |
| Tool callback registered on workflow agent | Callback is silently ignored | Tool callbacks are `LlmAgent`-only; move to the correct agent |
| Callback writes large data to state | Session bloat, slow persistence | Store large data as artifacts via `save_artifact()` instead of state |
| Plugin + agent callback conflict | Agent callback never executes | Plugin callbacks take precedence; remove duplicate logic from one layer |

---

## Scalability Notes

**Callback execution is synchronous within the agent loop.** Long-running callbacks (network calls, heavy computation) block the entire agent processing pipeline for that invocation. Keep callbacks fast — offload heavy work to background tasks or external queues if needed.

**State writes in callbacks accumulate in `state_delta`.** Frequent small writes across many callbacks increase the size of `state_delta` in events, which affects session persistence cost. Batch related state updates into a single callback where possible.

**Callback lists scale linearly.** Each callback in a list is invoked sequentially. A list of 10 callbacks adds 10x the per-callback latency. Use lists sparingly for genuinely independent concerns (logging, then guardrails, then context injection).

**`before_model_callback` frequency.** In a tool-using agent, a single user request can trigger 3-5+ LLM calls (initial, per tool result, summarization). Each fires `before_model_callback`. Caching logic should account for this multiplied invocation count.

**Plugin callbacks fire for every agent in the tree.** A `BasePlugin` `before_agent_callback` fires for the root agent, each sub-agent, and every workflow agent. In a tree with 10 agents, a single user request may trigger 10 `before_agent_callback` invocations from one plugin.

---

## Anti-Patterns

### 1. Monolithic Callback

Putting guardrails, logging, caching, and content filtering all in a single `before_model_callback`. This violates single-responsibility and makes testing difficult. Split into separate callbacks and use a callback list.

### 2. Returning Wrong Type

Returning `types.Content` from `before_model_callback` (should be `LlmResponse`) or `dict` from `before_agent_callback` (should be `types.Content`). Each hook has a specific override return type — mixing them up causes silent failures or TypeErrors.

### 3. Relying on Callback Propagation

Assuming a parent agent's callbacks automatically apply to sub-agents. They do not. Each agent in a multi-agent system needs its own callback registration. Create reusable callback functions and apply them explicitly to each agent.

### 4. Heavy I/O in Sync Callbacks

Performing database queries, HTTP calls, or file reads in synchronous callbacks. This blocks the event loop. Use `async` callbacks for any I/O operations.

### 5. Breaking Event Loop Early

Using `break` after `is_final_response()` in custom runner code. This prevents `after_agent_callback` from firing and can cause trace finalization issues in observability tools. Consume all events from the generator.

### 6. Duplicating Logic in Plugins and Agent Callbacks

Registering the same guardrail in both a `BasePlugin` and an agent's `before_model_callback`. The plugin fires first and may skip the agent callback, or both fire causing double-enforcement. Choose one layer for each concern.

### 7. Ignoring Cache Key Uniqueness

Using the same cache key pattern across different agents or tools. Cache keys should include the agent name or tool name to prevent cross-contamination: `f"temp:cache:{agent_name}:{hash(query)}"`.

### 8. Not Handling Multiple Model Calls

Writing a `before_model_callback` guardrail that sets a "checked" flag in state after the first invocation and skips checks on subsequent calls. The LLM is called multiple times per request; every call should be guarded.

---

## Guidelines

1. **Keep each callback focused on a single concern.** One callback for logging, one for guardrails, one for caching. Use callback lists to compose them on a single hook.

2. **Always wrap callback logic in `try/except`.** An unhandled exception in a callback can crash the agent invocation. Log the error and return `None` to fall through to default behavior.

3. **Use `temp:` prefix for transient callback state.** Cache keys, rate-limit counters, and per-request flags should use `temp:` prefix so they are not persisted across sessions. Use `app:` for global config and `user:` for user-specific data.

4. **Test callbacks in isolation with mock contexts.** Create `CallbackContext` mocks with controlled state to unit-test each callback function independently. Then integration-test with `InMemoryRunner`.

5. **Place logging callbacks before guardrail callbacks in lists.** Logging callbacks that return `None` always execute. If placed after a guardrail that may return non-`None`, the logging callback is skipped on blocked requests.

6. **Use Plugins for cross-cutting concerns, callbacks for agent-specific logic.** Logging, metrics, and global policy enforcement belong in Plugins. Agent-specific guardrails, caching, and prompt injection belong in agent callbacks.

7. **Document state keys your callbacks read and write.** Callbacks that interact with session state create implicit dependencies. Document which keys each callback reads, writes, and expects, especially in multi-agent systems where multiple agents share state.

8. **Use `async` callbacks for any I/O operations.** Database lookups, external API calls, and file operations should be in `async` callback functions. Sync callbacks should be reserved for pure in-memory operations (string checks, state reads, simple validation).

---

## References

1. [Callbacks Overview](https://google.github.io/adk-docs/callbacks/) — official ADK callbacks introduction, mechanism, and registration
2. [Types of Callbacks](https://google.github.io/adk-docs/callbacks/types-of-callbacks/) — detailed documentation for all six callback types with full code examples
3. [Callback Design Patterns and Best Practices](https://google.github.io/adk-docs/callbacks/design-patterns-and-best-practices/) — eight official design patterns and best practices
4. [Context Objects](https://google.github.io/adk-docs/context/) — `CallbackContext`, `ToolContext`, `ReadonlyContext` documentation
5. [Plugins](https://google.github.io/adk-docs/plugins/) — `BasePlugin` class, runner-level callbacks, plugin callback hooks
6. [Custom Tools — ToolContext](https://google.github.io/adk-docs/tools-custom/function-tools/) — `ToolContext` in tool functions, `actions`, `skip_summarization`
7. [LLM Agents](https://google.github.io/adk-docs/agents/llm-agents/) — `LlmAgent` parameters including all callback hooks
8. [Safety and Security](https://google.github.io/adk-docs/safety/) — callbacks and plugins for security guardrails
9. [Python API Reference](https://google.github.io/adk-docs/api-reference/python/google-adk.html) — `BaseAgent` callback type signatures, `Context` class
10. [Events](https://google.github.io/adk-docs/events/) — `Event.actions.state_delta`, event processing in the Runner loop
