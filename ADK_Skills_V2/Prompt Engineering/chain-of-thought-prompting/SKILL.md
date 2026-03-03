---
name: chain-of-thought-prompting
description: >
  Use this skill when a Google ADK LlmAgent needs to reason through multi-step
  problems explicitly before producing a final answer or tool call. Covers
  embedding chain-of-thought reasoning directives in the agent instruction,
  structuring step-by-step reasoning in agent outputs, using reasoning to
  improve tool call accuracy, and eliciting reliable reasoning in ADK agents.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: prompt-engineering
---

## Purpose

This skill implements chain-of-thought (CoT) prompting in Google ADK agents.
CoT prompting instructs the LLM to reason through a problem step-by-step before
producing a final answer or making a tool call, dramatically improving accuracy
on multi-step reasoning tasks, complex tool selection decisions, and structured
analysis workflows. In ADK, CoT reasoning is embedded in the `instruction` field
and shapes how the agent processes user queries before returning a response.

## When to Use

- When the agent must make complex multi-step decisions (e.g., routing a query
  to the right tool, diagnosing a system issue, solving a math or logic problem).
- When tool selection is ambiguous and the agent needs to explicitly reason about
  which tool to call and why.
- When the agent's accuracy on complex tasks is low and adding explicit reasoning
  steps would help.
- When the agent produces answers too quickly without adequate analysis.
- When users need to see the agent's reasoning, not just its conclusion.

## When NOT to Use

- Do not use CoT for simple, single-step queries — it adds unnecessary verbosity
  and latency.
- Do not use CoT when response brevity is mandatory (e.g., real-time chat).
- Do not force CoT for pure retrieval tasks — it doesn't improve factual lookup
  accuracy, only reasoning.

## Google ADK Context

- **`LlmAgent.instruction`**: CoT directives are embedded here. The instruction
  instructs the model to think step-by-step before answering or calling tools.
- **ADK Tool Call Reasoning**: Before making a function call, the LLM may emit
  a reasoning text block. CoT prompting makes this reasoning explicit and reliable.
- **`event.content.parts`**: In ADK, the LLM's intermediate reasoning can appear
  as text parts in events before the final function call or response part.
- **`output_schema`**: When combined with structured output, CoT reasoning can
  be captured in a `reasoning` field alongside the structured answer.
- **`before_agent_callback`**: Can inject a reasoning scaffold into state for
  use in CoT instruction templates.
- **ADK Agent Execution Lifecycle**: CoT reasoning happens during the LLM call
  phase. Each reasoning step may trigger additional tool calls in a ReAct-style
  loop.

## Capabilities

- Instructs ADK agents to reason step-by-step before tool calls and responses.
- Improves tool selection accuracy through explicit reasoning.
- Structures multi-step analysis in agent outputs.
- Enables ReAct-style (Reason + Act) agent behavior in ADK.
- Captures chain-of-thought in structured output schemas.
- Integrates reasoning verification into agent callbacks.

## Step-by-Step Instructions

1. **Add a CoT directive** to the `instruction` string:
   ```
   Before calling any tool or providing a final answer, think through the
   problem step by step. Structure your reasoning as:
   STEP 1: [Understand the user's request]
   STEP 2: [Identify what information is needed]
   STEP 3: [Decide which tool to call]
   STEP 4: [Verify the tool result]
   STEP 5: [Formulate the final answer]
   ```

2. **Specify reasoning visibility** if the user should see the thinking:
   ```
   Show your reasoning steps to the user before the final answer.
   ```
   Or if reasoning should be internal:
   ```
   Think through the steps internally. Show only the final answer to the user.
   ```

3. **Link CoT to tool selection** explicitly:
   ```
   Before calling a tool, explain WHY you are calling it and WHAT you expect
   to learn from it. After receiving the result, interpret it in the context
   of the user's original query.
   ```

4. **Apply structured CoT with output_schema** to capture reasoning formally:
   ```python
   from pydantic import BaseModel

   class ReasonedAnswer(BaseModel):
       reasoning_steps: list[str]
       tool_called: str | None
       final_answer: str

   agent = LlmAgent(
       name="reasoning_agent",
       model="gemini-2.0-flash",
       instruction=COT_INSTRUCTION,
       output_schema=ReasonedAnswer,
   )
   ```

5. **Validate reasoning quality** in `after_agent_callback` by checking that
   the response includes reasoning steps before the answer.

6. **Test with complex multi-step queries** and verify via Trace View that
   reasoning steps appear in the event stream.

## Input Format

```python
{
  "agent_goal": "Diagnose a customer complaint and route to the right tool",
  "user_input": "My order arrived damaged and I want a refund",
  "context": {"order_id": "ORD-12345", "user:name": "Alice"},
  "tools": ["check_order_status", "process_refund", "escalate_to_human"],
  "output_format": "reasoning_steps + final_action",
  "constraints": ["Always verify order before processing refund"]
}
```

## Output Format

```python
{
  "system_prompt": "Before calling any tool, think step by step...",
  "instructions": "...",
  "expected_output_format": {
    "reasoning_steps": ["Step 1: ...", "Step 2: ...", "Step 3: ..."],
    "final_answer": "string"
  },
  "tool_usage_guidance": {
    "check_order_status": "Call first to verify order details",
    "process_refund": "Call only after verifying order is valid"
  }
}
```

## Error Handling

- **Agent skips reasoning and jumps to answer**: Add "You MUST write at least
  3 reasoning steps before calling any tool or providing an answer."
- **Reasoning is circular or repetitive**: Add: "Each reasoning step must add
  new information. Do not repeat the same point twice."
- **CoT reasoning is too verbose**: Add an explicit limit: "Each reasoning step
  must be a single sentence of max 20 words."
- **Conflicting reasoning steps**: In the instruction, add: "If a reasoning step
  contradicts an earlier step, explicitly resolve the contradiction before
  proceeding."
- **Model ignores CoT instruction with simple queries**: That is correct behavior
  — for simple queries, the model may not need explicit reasoning. Only enforce
  CoT for complex, multi-step tasks.

## Examples

### Example 1 — ReAct-style agent with explicit CoT

```python
from google.adk.agents import LlmAgent
from my_tools import check_order_status, process_refund, escalate_to_human

COT_INSTRUCTION = """
You are a customer service agent for ShopBot.

REASONING PROTOCOL:
Before calling any tool or providing a final answer, always think step by step:
STEP 1: Understand what the customer needs.
STEP 2: Identify what information you need to confirm their request.
STEP 3: Decide which tool to call and explain why.
STEP 4: After receiving tool results, interpret the data.
STEP 5: Decide the best course of action.
STEP 6: Formulate a clear, empathetic response.

Show your reasoning steps to the customer with "[Thinking: ...]" prefix.
Follow with your final response without the "[Thinking]" prefix.

TOOLS:
- check_order_status: Call FIRST to verify order details before any action.
- process_refund: Call ONLY after confirming the order is eligible for refund.
- escalate_to_human: Call if the issue cannot be resolved automatically.
"""

agent = LlmAgent(
    name="customer_service_agent",
    model="gemini-2.0-flash",
    instruction=COT_INSTRUCTION,
    tools=[check_order_status, process_refund, escalate_to_human],
)
```

### Example 2 — Structured CoT output with Pydantic schema

```python
from google.adk.agents import LlmAgent
from pydantic import BaseModel
from typing import Optional

class DiagnosticResult(BaseModel):
    reasoning_steps: list[str]
    root_cause: str
    recommended_tool: Optional[str]
    final_response: str
    confidence: str  # "high", "medium", "low"

DIAGNOSTIC_INSTRUCTION = """
You are a technical support specialist for CloudSystems.

When a user reports an issue:
1. Analyze the symptoms systematically.
2. Reason through potential root causes.
3. Select the appropriate diagnostic tool.
4. Interpret tool results in context.
5. Provide a clear resolution.

Your response MUST follow the structured output schema. Fill all fields.
"""

agent = LlmAgent(
    name="diagnostic_agent",
    model="gemini-2.0-flash",
    instruction=DIAGNOSTIC_INSTRUCTION,
    output_schema=DiagnosticResult,
    tools=[run_diagnostic, check_system_logs, restart_service],
)
```

### Example 3 — CoT for tool selection disambiguation

```python
COT_TOOL_SELECTION_INSTRUCTION = """
You are a data analysis assistant with access to multiple data tools.

TOOL SELECTION PROTOCOL:
When the user asks a question, reason through tool selection as follows:
- Is this a SQL/structured data query? → use sql_query_tool
- Is this a real-time metric? → use metrics_api_tool
- Is this a document/text search? → use document_search_tool
- Is this ambiguous? → state your uncertainty and ask for clarification.

Before calling any tool, write:
"[Tool Selection Reasoning: I will use {tool_name} because {reason}]"
Then proceed with the tool call.
"""
```

## Edge Cases

- **CoT reasoning reveals ambiguous user intent**: The agent should ask for
  clarification rather than guessing. Add: "If Step 1 reveals ambiguity,
  ask one clarifying question before proceeding."
- **Very long CoT with many steps**: Cap at 6 numbered steps in the instruction.
  More steps cause the model to lose focus.
- **CoT with parallel tool needs**: If the analysis reveals two independent
  tools are needed, prefer `ParallelAgent` sub-agents rather than sequential
  CoT calls — it's more efficient.
- **CoT hidden from users** (internal reasoning): Use a `[INTERNAL REASONING]`
  delimiter in the instruction and implement `after_agent_callback` to strip
  it before returning the response.

## Integration Notes

- **`google-adk-structured-prompting`**: Combine CoT with structured output
  schemas to capture reasoning formally in a `reasoning_steps` field.
- **`google-adk-output-formatting`**: CoT output must still conform to the
  defined output format. Ensure reasoning steps are separate from the final answer.
- **`LlmAgent.output_schema`**: Use a Pydantic schema with a `reasoning_steps: list[str]`
  field to get CoT in a parseable format.
- **ADK Web UI Trace View**: The Trace tab shows the full model request and
  response per event — use this to verify that reasoning steps are being generated.
