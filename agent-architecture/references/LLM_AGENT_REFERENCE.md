# LLM Agent — Detailed Reference

## Agent Definition

An `LlmAgent` (aliased as `Agent`) uses a Large Language Model for reasoning, tool selection, and response generation. Its behavior is non-deterministic — the LLM dynamically decides the next action based on instructions and context.

## Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `name` | ✅ | Unique identifier. Avoid reserved name `user`. |
| `model` | ✅ | LLM model string (e.g. `gemini-2.5-flash`). |
| `description` | Recommended | Used by other agents for routing decisions. |
| `instruction` | Recommended | Core behavioral prompt. Supports `{state_var}` templates. |
| `tools` | Optional | List of functions, `BaseTool` subclasses, or `AgentTool` instances. |
| `sub_agents` | Optional | Child agents for delegation via transfer. |
| `output_key` | Optional | State key to persist the agent's final text response. |
| `input_schema` | Optional | Pydantic model defining expected input structure. |
| `output_schema` | Optional | Pydantic model defining expected output structure. |
| `include_contents` | Optional | Controls whether conversation history is sent to the LLM. |
| `generate_content_config` | Optional | Fine-tune LLM params (temperature, top_p, safety, etc.). |
| `planner` | Optional | Enable planning mode for multi-step reasoning. |
| `code_executor` | Optional | Enable code execution capabilities. |

## Code Examples

### Basic Agent

```python
from google.adk.agents import LlmAgent

support_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="support_agent",
    description="Handles customer support inquiries about orders and returns.",
    instruction="""You are a helpful customer support agent.
When a user asks about their order:
1. Ask for their order ID if not provided.
2. Use the `lookup_order` tool to retrieve order details.
3. Provide a clear, friendly summary of the order status.

Always be polite and professional.""",
    tools=[lookup_order],
)
```

### Agent with State Templates

```python
greeting_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="greeting_agent",
    instruction="Greet the user by name: {user_name}. Their preferred language is {language?}.",
    output_key="greeting_message",
)
# {user_name} is required in state; {language?} is optional (no error if missing).
```

### Agent with Output Schema

```python
from pydantic import BaseModel

class MovieReview(BaseModel):
    title: str
    rating: float
    summary: str

review_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="review_agent",
    instruction="Analyze the given movie and produce a structured review.",
    output_schema=MovieReview,
    output_key="review_data",
)
```

### Tool Definition

```python
def lookup_order(order_id: str) -> dict:
    """Retrieves order details for a given order ID.

    Args:
        order_id: The unique identifier for the customer's order.

    Returns:
        A dictionary with order status, items, and delivery date.
    """
    # Replace with actual database/API lookup
    orders = {
        "ORD-001": {"status": "shipped", "items": ["Book", "Pen"], "delivery": "2025-03-15"},
        "ORD-002": {"status": "processing", "items": ["Laptop"], "delivery": "2025-03-20"},
    }
    return orders.get(order_id, {"error": f"Order {order_id} not found."})

# Pass directly to the agent — ADK auto-wraps it as a FunctionTool
support_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="support_agent",
    instruction="...",
    tools=[lookup_order],
)
```

## Advanced Configuration

### Controlling LLM Generation

```python
from google.genai import types

agent = LlmAgent(
    model="gemini-2.5-flash",
    name="creative_writer",
    instruction="Write creative short stories.",
    generate_content_config=types.GenerateContentConfig(
        temperature=0.9,
        top_p=0.95,
        max_output_tokens=2048,
    ),
)
```

### Disabling Conversation History

```python
stateless_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="stateless_classifier",
    instruction="Classify the input text into one of: positive, negative, neutral.",
    include_contents="none",  # Don't send prior conversation turns
)
```

## Key Guidelines

1. **Instructions are the #1 lever** — invest time writing clear, specific instructions with examples.
2. **Tool docstrings matter** — the LLM reads function names, docstrings, and parameter types to decide when/how to call tools.
3. **Use `output_key`** to chain agents — downstream agents read state via `{key}` in instructions.
4. **Description is for routing** — in multi-agent setups, sibling agents are discovered via their descriptions.
