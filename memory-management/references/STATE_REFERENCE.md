# State Management — Detailed Reference

## Overview

`session.state` is a key-value dictionary that acts as the agent's scratchpad during a conversation. Values must be serializable (strings, numbers, booleans, simple lists/dicts).

## State Prefixes

### Session State (No Prefix)

```python
# Scoped to the current session only
ctx.state["current_step"] = "confirm_payment"
ctx.state["selected_product"] = "Premium Plan"
```

### User State (`user:` prefix)

```python
# Shared across all sessions for this user
ctx.state["user:preferred_language"] = "fr"
ctx.state["user:timezone"] = "America/New_York"
ctx.state["user:name"] = "Alice"
```

### App State (`app:` prefix)

```python
# Shared across all users and sessions
ctx.state["app:global_discount_code"] = "SAVE10"
ctx.state["app:maintenance_mode"] = False
```

### Temporary State (`temp:` prefix)

```python
# Discarded after the current invocation completes
ctx.state["temp:raw_api_response"] = {"status": 200, "data": [...]}
ctx.state["temp:intermediate_calculation"] = 42.5
```

## Updating State — The Right Way

### Method 1: `output_key` (Simplest)

Automatically saves the agent's text response into state:

```python
from google.adk.agents import LlmAgent

summary_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="summarizer",
    instruction="Summarize the user's request in one sentence.",
    output_key="request_summary",  # Response auto-saved to state["request_summary"]
)
```

### Method 2: ToolContext (Inside Tools)

```python
def process_order(order_id: str, tool_context) -> dict:
    """Process an order and update state with results."""
    # Simulate processing
    result = {"status": "confirmed", "total": 99.99}

    # Update state through tool_context — tracked by event system
    tool_context.state["order_status"] = result["status"]
    tool_context.state["order_total"] = result["total"]
    tool_context.state["temp:processing_time_ms"] = 150

    return result
```

### Method 3: CallbackContext (Inside Callbacks)

```python
async def track_interactions(callback_context):
    """After-agent callback that counts interactions."""
    count = callback_context.state.get("interaction_count", 0)
    callback_context.state["interaction_count"] = count + 1
    callback_context.state["temp:last_action"] = "agent_completed"
```

### ❌ WRONG: Direct Session Modification

```python
# DO NOT DO THIS — bypasses event tracking, breaks persistence
session = await session_service.get_session(app_name="app", user_id="u1", session_id="s1")
session.state["my_key"] = "value"  # ❌ Not tracked, not persisted, not thread-safe
```

## Reading State in Instructions

Use `{key}` template syntax in agent instructions:

```python
from google.adk.agents import LlmAgent, SequentialAgent

# Agent 1: Saves output to state
collector = LlmAgent(
    model="gemini-2.5-flash",
    name="data_collector",
    instruction="Find the current stock price of the requested company.",
    output_key="stock_price",
)

# Agent 2: Reads state via template
analyst = LlmAgent(
    model="gemini-2.5-flash",
    name="analyst",
    instruction="Write a brief analysis for a stock currently priced at {stock_price}.",
    output_key="analysis",
)

pipeline = SequentialAgent(name="stock_pipeline", sub_agents=[collector, analyst])
```

### Optional State Variables

Append `?` to avoid errors when a state variable might not exist:

```python
agent = LlmAgent(
    model="gemini-2.5-flash",
    name="greeting_agent",
    instruction="Greet the user. Their name is {user:name?}. Preferred language: {user:language?}.",
)
# If user:name or user:language don't exist in state, no error is raised
```

## Full Example: Stateful Multi-Agent Workflow

```python
from google.adk.agents import LlmAgent, SequentialAgent
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.genai.types import Content, Part

# Define agents with state passing
intake_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="intake_agent",
    instruction="Extract the customer's issue category from their message. Categories: billing, shipping, technical.",
    output_key="issue_category",
)

resolver_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="resolver_agent",
    instruction="""You are a support agent. The customer's issue category is: {issue_category}.
Provide a helpful resolution for their problem.""",
    output_key="resolution",
)

# Chain them
support_pipeline = SequentialAgent(
    name="support_pipeline",
    sub_agents=[intake_agent, resolver_agent],
)

# Setup and run
session_service = InMemorySessionService()
runner = Runner(agent=support_pipeline, app_name="support_app", session_service=session_service)

async def handle_request():
    session = await session_service.create_session(
        app_name="support_app", user_id="customer_1", session_id="ticket_001"
    )

    user_msg = Content(parts=[Part(text="My package hasn't arrived yet")], role="user")
    async for event in runner.run_async(
        user_id="customer_1", session_id="ticket_001", new_message=user_msg
    ):
        if event.is_final_response() and event.content and event.content.parts:
            print(f"Resolution: {event.content.parts[0].text}")

    # Check final state
    updated = await session_service.get_session(
        app_name="support_app", user_id="customer_1", session_id="ticket_001"
    )
    print(f"Category: {updated.state.get('issue_category')}")
    print(f"Resolution: {updated.state.get('resolution')}")
```

## Best Practices Summary

1. **Minimize state** — only store essential, dynamic data
2. **Use prefixes correctly** — `user:` for cross-session user data, `temp:` for throwaway data
3. **Never store complex objects** — only serializable types
4. **Always update through context** — `ToolContext.state` or `CallbackContext.state`
5. **Use `output_key`** for simple agent-to-agent data passing
6. **Use `{key?}` for optional variables** — prevents errors when state may not exist
