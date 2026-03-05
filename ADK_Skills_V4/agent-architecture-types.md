---
name: agent-architecture-types
description: >
  Comprehensive reference for agent architecture and agent types in Google Agent Development
  Kit (ADK) Python. Covers LLM Agent Fundamentals (`LlmAgent`, `Agent` alias, `instruction`,
  `model`, `tools`, `output_key`, `generate_content_config`, `input_schema`, `output_schema`,
  `static_instruction`, `planner`, `include_contents`), Multi-Agent Systems (agent hierarchy,
  `sub_agents`, LLM-driven delegation via `transfer_to_agent`, `AgentTool` explicit invocation,
  `global_instruction`, `disallow_transfer_to_parent`, `disallow_transfer_to_peers`, shared
  session state communication), Sequential Workflow Agents (`SequentialAgent`, deterministic
  pipelines, data passing via `output_key` and `{key}` templating), Parallel Workflow Agents
  (`ParallelAgent`, concurrent execution, fan-out/gather pattern, unique state keys),
  Loop Workflow Agents (`LoopAgent`, `max_iterations`, `escalate=True` termination, `exit_loop`
  tool pattern), Custom Agents (subclassing `BaseAgent`, `_run_async_impl`, `_run_live_impl`,
  `InvocationContext`, custom orchestration, yielding `Event` objects), and Agent Configuration
  (`GenerateContentConfig`, `temperature`, `max_output_tokens`, `top_p`, `top_k`,
  `safety_settings`, structured output with Pydantic `BaseModel`). Consult when designing
  agent systems, choosing between agent types, configuring LLM behavior, building multi-agent
  hierarchies, or implementing workflow orchestration patterns.
metadata:
  author: adk-skills
  version: "1.0"
---

# Agent Architecture & Types — Detailed Reference

## Overview

This skill covers all agent types and architectural patterns in Google ADK (Python). The agent system forms the core of every ADK application, determining how intelligence is structured, how tasks are distributed, and how execution flows:

1. **LLM Agent Fundamentals** — `LlmAgent` (aliased as `Agent`) powered by Large Language Models with instructions, tools, model configuration, and structured I/O.
2. **Multi-Agent Systems** — hierarchies of cooperating agents using LLM-driven delegation, `AgentTool` explicit invocation, and shared session state for communication.
3. **Sequential Workflow Agents** — `SequentialAgent` for deterministic pipelines that execute sub-agents one after another in fixed order.
4. **Parallel Workflow Agents** — `ParallelAgent` for concurrent execution of independent sub-agents with fan-out/gather patterns.
5. **Loop Workflow Agents** — `LoopAgent` for iterative refinement cycles with `max_iterations` and `escalate`-based termination.
6. **Custom Agents** — subclassing `BaseAgent` with `_run_async_impl` / `_run_live_impl` for arbitrary orchestration logic.
7. **Agent Configuration** — `GenerateContentConfig` for fine-grained LLM control (temperature, tokens, safety) and schema-based structured output.

All agent types inherit from `BaseAgent`. Workflow agents (`SequentialAgent`, `ParallelAgent`, `LoopAgent`) are deterministic — they do not use an LLM for orchestration decisions, only for the internal logic of their sub-agents. `LlmAgent` is non-deterministic — it uses the LLM to decide which tools to call, which sub-agent to transfer to, and how to respond.

For session state management and state prefixes, see `sessions-state-memory.md`.
For tool definitions and integrations, see `tools-and-integrations.md`.
For context objects, events, and artifacts, see `context-and-events.md`.

---

## Prerequisites

- Python 3.10+
- `pip install google-adk`
- A Gemini model string (e.g., `"gemini-2.0-flash"`, `"gemini-2.5-flash"`) or a `LiteLlm`-wrapped model for non-Gemini providers
- For structured output with tools: Gemini 3.0+ (using `output_schema` with `tools` in the same request is only supported on Gemini 3.0+)
- For Vertex AI deployment: `GOOGLE_GENAI_USE_VERTEXAI=TRUE`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION` env vars
- For non-Gemini models: `pip install litellm` and use `from google.adk.models.lite_llm import LiteLlm`

---

## Core Concept

### Agent Type Hierarchy

ADK provides three categories of agents, all inheriting from `BaseAgent`:

```
                         BaseAgent
                    (name, description,
                     sub_agents,
                     before_agent_callback,
                     after_agent_callback)
                            │
            ┌───────────────┼───────────────┐
            │               │               │
        LlmAgent      Workflow Agents   Custom Agents
     (Agent alias)         │            (your subclass)
     Non-deterministic     │            _run_async_impl
     LLM-powered           │            _run_live_impl
                    ┌──────┼──────┐
                    │      │      │
              Sequential Parallel Loop
               Agent     Agent   Agent
              (in-order) (concurrent) (iterative)
```

**LlmAgent** uses an LLM to reason about instructions, decide which tools to call, and whether to transfer control to another agent. It is the primary "thinking" agent.

**Workflow Agents** are deterministic orchestrators. They control *when* sub-agents run (in sequence, in parallel, or in a loop) but do not use an LLM for the flow control itself. The sub-agents they orchestrate *may* be LLM-powered.

**Custom Agents** inherit from `BaseAgent` and implement `_run_async_impl` directly, giving complete control over orchestration logic including conditional branching, dynamic agent selection, and external API integration within the control flow.

### LlmAgent Parameter Map

The `LlmAgent` class (importable as `Agent`) has these key parameters:

```
LlmAgent(
    name: str,                          # Required. Unique identifier.
    model: str | BaseLlm,               # Required. Model string or LiteLlm wrapper.
    instruction: str | Callable,        # Agent-specific instructions. Supports {key} templating.
    description: str,                   # Used by parent agents for routing decisions.
    tools: list,                        # FunctionTool, BaseTool, AgentTool instances.
    sub_agents: list[BaseAgent],        # Child agents for delegation.
    output_key: str | None,             # Saves final response text to session.state[output_key].
    generate_content_config: GenerateContentConfig,  # LLM parameters.
    input_schema: type[BaseModel],      # Pydantic model for structured input.
    output_schema: type[BaseModel],     # Pydantic model for structured output.
    global_instruction: str | Callable, # Prepended to ALL agents in the tree (set on root).
    static_instruction: str,            # Cacheable instruction content (for context caching).
    include_contents: str,              # 'default' or 'none'. Controls conversation history.
    planner: BasePlanner,               # Enables multi-step reasoning (BuiltInPlanner, PlanReActPlanner).
    disallow_transfer_to_parent: bool,  # Prevents transferring back to parent. Default: False.
    disallow_transfer_to_peers: bool,   # Prevents transferring to sibling agents. Default: False.
    before_agent_callback: Callable,    # Runs before _run_async_impl.
    after_agent_callback: Callable,     # Runs after _run_async_impl.
    before_model_callback: Callable,    # Runs before each LLM call.
    after_model_callback: Callable,     # Runs after each LLM call.
    before_tool_callback: Callable,     # Runs before each tool execution.
    after_tool_callback: Callable,      # Runs after each tool execution.
)
```

### Multi-Agent Communication Mechanisms

ADK provides three ways for agents to communicate:

```
┌─────────────────────────────────────────────────────────────────┐
│                  Communication Mechanisms                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Shared Session State (whiteboard)                            │
│     Agent A writes → state["result_a"] → Agent B reads          │
│     Used by: output_key, {key} templating, ToolContext.state     │
│                                                                  │
│  2. LLM-Driven Delegation (transfer)                             │
│     Parent LLM decides → transfer_to_agent("SubAgent")           │
│     Based on: sub_agent descriptions + parent instruction        │
│     Transfer scope: parent, children, siblings (configurable)    │
│                                                                  │
│  3. Explicit Invocation (AgentTool)                              │
│     Parent calls agent as tool → AgentTool(agent=specialist)     │
│     Returns result to parent → parent retains control            │
│     Difference: sub_agent transfer = full handoff;               │
│                 AgentTool = function call (parent stays in loop)  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Workflow Agent Execution Models

```
SequentialAgent: [A] → [B] → [C]          (pipeline)
ParallelAgent:   [A] ┬ [B] ┬ [C]          (concurrent)
LoopAgent:       [A → B → C] × N          (iterative, until escalate or max_iterations)
```

All three workflow agent types share the `InvocationContext` with their sub-agents, meaning sub-agents access the same `session.state` (including `temp:` namespace). Data flows between steps primarily through `output_key` writes and `{key}` instruction templating reads.

---

## When to Use

| Scenario | Agent Type | Why |
|----------|-----------|-----|
| Single conversational assistant | `LlmAgent` | Core reasoning with tools and natural language |
| Customer support routing | `LlmAgent` with `sub_agents` | LLM-driven delegation to specialists |
| ETL / data pipeline | `SequentialAgent` | Fixed order, deterministic stages |
| Multi-source research | `ParallelAgent` inside `SequentialAgent` | Independent lookups then synthesis |
| Iterative code review | `LoopAgent` | Refine until quality check passes |
| Conditional branching logic | Custom `BaseAgent` subclass | Runtime if/else on state values |
| Agent-as-a-function | `AgentTool` wrapping an agent | Parent calls specialist, retains control |
| Deterministic + LLM hybrid | `SequentialAgent` containing `LlmAgent` children | Workflow controls flow, LLMs provide intelligence |

---

## Rules

1. **Every agent must have a unique `name`** within the agent tree. The framework uses names for transfer routing and event attribution.

2. **`Agent` is an alias for `LlmAgent`**. Both are imported from `google.adk.agents`:
   ```python
   from google.adk.agents import Agent       # alias
   from google.adk.agents import LlmAgent    # same class
   ```

3. **Workflow agents do not accept a `model` parameter**. They have no LLM — only `sub_agents`:
   ```python
   from google.adk.agents import SequentialAgent
   pipeline = SequentialAgent(name="Pipeline", sub_agents=[agent_a, agent_b])
   # No 'model' or 'instruction' parameter
   ```

4. **`output_key` writes the agent's final text response to `session.state[key]`**. This is the primary mechanism for passing data between agents in workflow pipelines. In Python: `session.state[output_key] = agent_response_text`.

5. **`{key}` templating in `instruction` reads from `session.state`**. Use `{key?}` (with `?`) for optional keys that may not exist yet — this prevents `KeyError` if the key is missing.

6. **`sub_agents` creates a parent-child hierarchy**. An agent can only have one parent. When you set `sub_agents=[a, b]`, agents `a` and `b` have their `parent_agent` set to the parent.

7. **LLM-driven delegation uses `transfer_to_agent`**. The LLM generates a `FunctionCall(name='transfer_to_agent', args={'agent_name': '...'})` based on sub-agent `description` fields. Clear, distinct descriptions are vital for correct routing.

8. **`AgentTool` wraps an agent as a callable tool**. Unlike sub-agent transfer (full handoff), `AgentTool` returns results to the calling agent, which retains control:
   ```python
   from google.adk.tools.agent_tool import AgentTool
   specialist_tool = AgentTool(agent=specialist_agent)
   parent = LlmAgent(name="Parent", tools=[specialist_tool], ...)
   ```

9. **`LoopAgent` terminates on `escalate=True` or `max_iterations`**. The `escalate` signal propagates upward — it also stops parent `SequentialAgent` or other workflow agents above the loop. To exit a loop without halting parent workflows, use a custom `BaseAgent` that conditionally yields `Event(actions=EventActions(escalate=True))`.

10. **`escalate=True` propagates up the entire agent tree**. When a sub-agent sets `escalate=True` (via tool or event), it terminates the loop *and* any parent workflow agents. This is a known behavior — subsequent agents in a parent `SequentialAgent` will not run after a loop exits via escalation.

11. **`ParallelAgent` sub-agents share session state**. Each parallel sub-agent should write to a unique `output_key` to prevent race conditions. Do not have parallel agents write to the same state key.

12. **Custom agents must yield at least one `Event`** from `_run_async_impl`. The method signature is `async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]`.

13. **`global_instruction` on the root agent is prepended to every agent in the tree**. Set it only on the root agent — it applies to all descendants automatically.

14. **`disallow_transfer_to_parent` and `disallow_transfer_to_peers` influence the LLM prompt**, guiding which `transfer_to_agent` targets are presented to the model. Note: these are prompt-level controls, not hard enforcement — the LLM may still attempt transfers if it infers agent names from conversation history.

---

## Example

A complete multi-agent system combining LLM agents, workflow agents, and configuration:

```python
import asyncio
from google.adk.agents import LlmAgent, SequentialAgent, ParallelAgent, LoopAgent, BaseAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events import Event, EventActions
from google.adk.tools.agent_tool import AgentTool
from google.adk.tools.tool_context import ToolContext
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types
from typing import AsyncGenerator
from pydantic import BaseModel, Field

MODEL = "gemini-2.0-flash"

# --- Structured output schema ---
class ReviewResult(BaseModel):
    quality: str = Field(description="'pass' or 'fail'")
    feedback: str = Field(description="Specific improvement suggestions")

# --- Tool for exiting the loop ---
def exit_loop(tool_context: ToolContext):
    """Call when the review quality is 'pass', signaling refinement is complete."""
    tool_context.actions.escalate = True
    tool_context.actions.skip_summarization = True
    return {}

# --- LLM Agents ---
researcher_energy = LlmAgent(
    name="EnergyResearcher",
    model=MODEL,
    instruction="Research the latest advancements in renewable energy. Summarize in 2-3 sentences.",
    description="Researches renewable energy sources.",
    output_key="energy_findings",
)

researcher_ev = LlmAgent(
    name="EVResearcher",
    model=MODEL,
    instruction="Research the latest advancements in electric vehicles. Summarize in 2-3 sentences.",
    description="Researches electric vehicle technology.",
    output_key="ev_findings",
)

draft_writer = LlmAgent(
    name="DraftWriter",
    model=MODEL,
    instruction="""Write a research report combining these findings:
    - Energy: {energy_findings}
    - EVs: {ev_findings}
    Output a structured report with sections for each topic.""",
    description="Combines parallel research into a draft report.",
    output_key="current_draft",
)

reviewer = LlmAgent(
    name="Reviewer",
    model=MODEL,
    instruction="""Review the draft: {current_draft}
    If quality is acceptable, call the exit_loop tool.
    Otherwise, provide specific feedback for improvement.""",
    description="Reviews draft quality and exits loop when satisfied.",
    tools=[exit_loop],
    output_key="review_feedback",
)

refiner = LlmAgent(
    name="Refiner",
    model=MODEL,
    instruction="""Improve this draft based on feedback:
    Draft: {current_draft}
    Feedback: {review_feedback}
    Output only the improved draft.""",
    description="Refines draft based on review feedback.",
    output_key="current_draft",
)

# --- Workflow Agents ---
parallel_research = ParallelAgent(
    name="ParallelResearch",
    sub_agents=[researcher_energy, researcher_ev],
)

refinement_loop = LoopAgent(
    name="RefinementLoop",
    sub_agents=[reviewer, refiner],
    max_iterations=3,
)

pipeline = SequentialAgent(
    name="ResearchPipeline",
    sub_agents=[parallel_research, draft_writer, refinement_loop],
    description="Full research pipeline: parallel gather → draft → iterative refinement.",
)

root_agent = pipeline

# --- Run the pipeline ---
async def main():
    session_service = InMemorySessionService()
    runner = Runner(agent=root_agent, app_name="research_app", session_service=session_service)
    session = await session_service.create_session(app_name="research_app", user_id="user1")

    user_message = types.Content(role="user", parts=[types.Part(text="Research clean energy trends")])
    async for event in runner.run_async(
        user_id="user1", session_id=session.id, new_message=user_message
    ):
        if event.is_final_response() and event.content and event.content.parts:
            print(event.content.parts[0].text)

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Decision Table

| Question | Answer | Action |
|----------|--------|--------|
| Does the agent need LLM reasoning? | Yes | Use `LlmAgent` |
| Does the agent need LLM reasoning? | No, deterministic flow | Use workflow agent or custom `BaseAgent` |
| Should steps run in fixed order? | Yes | `SequentialAgent` |
| Should steps run concurrently? | Yes, independent tasks | `ParallelAgent` |
| Should steps repeat until a condition? | Yes | `LoopAgent` with `max_iterations` + escalation |
| Need conditional branching at runtime? | Yes | Custom `BaseAgent` with `_run_async_impl` |
| Should parent retain control after calling a specialist? | Yes | Wrap specialist in `AgentTool`, add to `tools` |
| Should specialist fully take over the conversation? | Yes | Add specialist to `sub_agents`, let LLM delegate |
| Need system-wide instructions? | Yes | Set `global_instruction` on the root `LlmAgent` |
| Need to prevent agent from returning to parent? | Yes | Set `disallow_transfer_to_parent=True` |
| Need deterministic output format? | Yes | Set `output_schema` with a Pydantic `BaseModel` |
| Need to control LLM randomness? | Yes | Set `generate_content_config` with `temperature` |
| Need to run agents after a loop exits? | Yes | Beware: `escalate` propagates up — consider custom agent |

---

## Common Patterns

### Hub-and-Spoke Router

A central coordinator agent delegates to specialists based on user intent:

```python
billing = LlmAgent(name="Billing", model=MODEL, description="Handles billing and payment inquiries.")
support = LlmAgent(name="Support", model=MODEL, description="Handles technical support requests.")

coordinator = LlmAgent(
    name="HelpDesk",
    model=MODEL,
    instruction="Route user requests: billing issues to Billing, technical problems to Support.",
    description="Main help desk router.",
    sub_agents=[billing, support],
)
```

The coordinator's LLM reads sub-agent `description` fields and generates `transfer_to_agent(agent_name='Billing')` or `transfer_to_agent(agent_name='Support')` as appropriate. This is called **LLM-driven delegation** or **AutoFlow**.

### Fan-Out / Gather

Parallel research followed by synthesis:

```python
gather = ParallelAgent(name="Gather", sub_agents=[agent_a, agent_b, agent_c])
synthesizer = LlmAgent(
    name="Synthesizer",
    model=MODEL,
    instruction="Combine results from {result_a}, {result_b}, {result_c}.",
)
pipeline = SequentialAgent(name="FanOutGather", sub_agents=[gather, synthesizer])
```

Each parallel sub-agent writes to a unique `output_key`. The synthesizer reads all keys via `{key}` templating.

### Iterative Refinement with Exit Tool

A critic-refiner loop that exits when quality is acceptable:

```python
def exit_loop(tool_context: ToolContext):
    """Call when critique indicates no further changes needed."""
    tool_context.actions.escalate = True
    return {}

critic = LlmAgent(name="Critic", model=MODEL, instruction="Critique {current_draft}.", output_key="criticism")
refiner = LlmAgent(
    name="Refiner", model=MODEL,
    instruction="Improve draft based on {criticism}. If no issues, call exit_loop.",
    tools=[exit_loop], output_key="current_draft",
)
loop = LoopAgent(name="RefineLoop", sub_agents=[critic, refiner], max_iterations=5)
```

### Agent-as-Tool (Explicit Invocation)

Parent calls a specialist agent like a function and retains control:

```python
from google.adk.tools.agent_tool import AgentTool

summarizer = LlmAgent(name="Summarizer", model=MODEL, description="Summarizes text.")
summary_tool = AgentTool(agent=summarizer)

report_writer = LlmAgent(
    name="ReportWriter",
    model=MODEL,
    instruction="Write a report on AI trends. Use the Summarizer tool for condensing research.",
    tools=[summary_tool],
)
```

Key difference: with `sub_agents`, control transfers fully to the sub-agent. With `AgentTool`, the result returns to the parent agent, which continues its own reasoning.

### Custom Conditional Orchestrator

A `BaseAgent` subclass with runtime branching:

```python
class ConditionalRouter(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext) -> AsyncGenerator[Event, None]:
        sentiment = ctx.session.state.get("sentiment", "neutral")
        if sentiment == "negative":
            target = self.sub_agents[0]  # escalation_agent
        else:
            target = self.sub_agents[1]  # standard_agent
        async for event in target.run_async(ctx):
            yield event
```

---

## Edge Cases

### 1. `escalate` Propagation Through Parent Workflow Agents
When a sub-agent inside a `LoopAgent` sets `escalate=True`, the signal propagates upward through all parent workflow agents. If the `LoopAgent` is nested inside a `SequentialAgent`, subsequent agents in the sequence will **not** run. Workaround: use a custom `BaseAgent` that checks a state flag instead of relying on `escalate` for loop termination, or restructure the pipeline so no agents need to run after the loop.

### 2. `output_schema` with Tools Requires Gemini 3.0+
Using `output_schema` and `tools` on the same `LlmAgent` is only supported by Gemini 3.0 and above. On earlier models, the LLM may fail to produce structured output while also calling tools. Workaround: use a `SequentialAgent` where one sub-agent calls tools and another (with `output_schema`, no tools) formats the output.

### 3. `{key}` Templating with Missing State Keys
If an instruction contains `{some_key}` and `session.state["some_key"]` does not exist, the framework raises a `KeyError`. Use `{some_key?}` (trailing `?`) to make it optional — the placeholder is replaced with an empty string if the key is absent.

### 4. Parallel Agents Writing to the Same State Key
If two `ParallelAgent` sub-agents use the same `output_key`, a race condition occurs. The last agent to finish overwrites the other's result. Always assign unique `output_key` values to each parallel sub-agent.

### 5. `disallow_transfer_to_peers` Is Prompt-Level Only
Setting `disallow_transfer_to_peers=True` modifies the system prompt to discourage peer transfers, but it is not enforced at the framework level. If the LLM infers sibling agent names from conversation history, it may still attempt a transfer. For hard enforcement, use a `before_tool_callback` to intercept and reject `transfer_to_agent` calls to unauthorized targets.

### 6. `LlmAgent` Without a Model
If you create an `LlmAgent` without specifying `model`, it uses `LlmAgent.DEFAULT_MODEL`. In workflow agents (`SequentialAgent`, `ParallelAgent`, `LoopAgent`), no model is needed — they are modelless orchestrators. Attempting to set `model` on a workflow agent has no effect.

### 7. Custom Agent Not Yielding Any Events
A custom `BaseAgent` subclass must yield at least one `Event` from `_run_async_impl`. If the method returns without yielding, the Runner receives no output and the invocation appears to hang or produce no response.

### 8. `AgentTool` and Multimodal Responses
`AgentTool` converts the wrapped agent's final response into a text-based tool result for the calling LLM. If the wrapped agent produces multimodal content (images, audio), the `AgentTool` may not propagate it correctly. For multimodal scenarios, use `sub_agents` transfer instead of `AgentTool`.

### 9. An Agent Added as Sub-Agent Twice
An agent instance can only have one parent. If you add the same agent instance to two different parents' `sub_agents`, the second assignment overwrites the first parent. To reuse identical agent logic in two places, create two separate instances with different `name` values.

### 10. `LoopAgent` Without `max_iterations` or Escalation
If neither `max_iterations` is set nor any sub-agent triggers `escalate=True`, the loop runs indefinitely. Always set `max_iterations` as a safety net, even when using escalation-based exit.

---

## Failure Handling

| Failure | Symptom | Resolution |
|---------|---------|------------|
| `KeyError` in instruction templating | Agent crashes on startup | Use `{key?}` instead of `{key}` for optional state |
| Loop never terminates | Infinite loop, escalating token costs | Set `max_iterations`; verify escalation tool is reachable |
| `escalate` kills parent pipeline | Agents after loop never execute | Use custom BaseAgent checker or restructure pipeline |
| Wrong agent receives transfer | Coordinator routes to incorrect specialist | Improve sub-agent `description` fields; make them distinct |
| `output_schema` produces malformed JSON | LLM ignores schema constraint | Ensure model supports structured output; add explicit instruction |
| Parallel race condition on state | Inconsistent or lost data | Use unique `output_key` per parallel sub-agent |
| Custom agent hangs | No events yielded | Ensure `_run_async_impl` yields at least one `Event` |
| `AgentTool` drops multimodal content | Images/audio lost in tool result | Use `sub_agents` transfer for multimodal responses |
| Transfer to nonexistent agent | Framework error or silent failure | Verify all agent names match exactly (case-sensitive) |
| `generate_content_config` ignored | LLM uses default parameters | Confirm the parameter is set on the correct `LlmAgent`, not on workflow agent |

---

## Scalability Notes

- **Nesting depth**: Workflow agents can be nested arbitrarily (e.g., `SequentialAgent` containing a `ParallelAgent` containing `LoopAgent` children). Keep nesting to 3-4 levels for readability and debuggability.
- **Parallel concurrency**: `ParallelAgent` launches all sub-agents concurrently using Python's `asyncio`. The number of parallel sub-agents is bounded by available model API rate limits, not by the framework.
- **Loop iterations**: Each loop iteration makes at minimum one LLM call per LLM-powered sub-agent. With `max_iterations=10` and 3 LLM sub-agents, expect ~30 LLM calls in the worst case. Budget token costs accordingly.
- **Agent tree size**: There is no hard limit on the number of agents in a tree, but each `LlmAgent`'s system prompt includes descriptions of all transferable agents. Large agent pools (20+ siblings) can bloat the system prompt and degrade routing accuracy.
- **State size**: All workflow agents share `InvocationContext` and therefore share `session.state`. Large state values (e.g., full documents stored via `output_key`) increase memory usage across all agents in the pipeline.
- **`global_instruction` overhead**: This text is prepended to every LLM call in the entire agent tree. Keep it concise (1-2 paragraphs) to avoid wasting tokens on every agent invocation.

---

## Anti-Patterns

### 1. Using `SequentialAgent` When Order Does Not Matter
If sub-agents are independent and don't depend on each other's output, use `ParallelAgent` for speed. `SequentialAgent` forces unnecessary serialization.

### 2. Storing Large Documents in `output_key`
`output_key` stores the entire text response in `session.state`. For large outputs (full reports, codebases), this bloats state and increases token usage for subsequent agents reading it via `{key}` templating. Consider storing references or summaries instead.

### 3. Relying on `disallow_transfer_to_peers` for Security
These flags are prompt-level hints, not hard enforcement. Do not use them as security boundaries. Use `before_tool_callback` to intercept and validate `transfer_to_agent` calls for access control.

### 4. `LoopAgent` Without `max_iterations`
Never deploy a `LoopAgent` without `max_iterations`. If the exit condition is never met (LLM doesn't call the exit tool, or state flag is never set), the loop runs forever, burning tokens and compute.

### 5. Using `AgentTool` When Full Conversation Context Is Needed
`AgentTool` does not pass full conversation history to the wrapped agent. If the specialist needs multi-turn context, use `sub_agents` delegation instead, where the sub-agent receives the full `InvocationContext` including session history.

### 6. Giving All Agents the Same Description
The LLM uses `description` fields to decide which sub-agent to delegate to. If descriptions are vague or overlapping, routing becomes unreliable. Each agent should have a unique, specific description that clearly differentiates its domain.

### 7. Putting All Logic in a Single LlmAgent
A monolithic agent with dozens of tools and a massive instruction string is hard to debug, test, and maintain. Break complex logic into specialized sub-agents with focused responsibilities.

### 8. Custom BaseAgent Without Error Handling
If `_run_async_impl` raises an unhandled exception, the entire invocation fails. Wrap orchestration logic in try/except and yield an error `Event` so the Runner can handle it gracefully.

---

## Guidelines

1. **Start with a single `LlmAgent`**, then decompose into multi-agent only when complexity demands it. Premature decomposition adds overhead without benefit.

2. **Use `description` as API documentation for the LLM**. When using LLM-driven delegation, the parent's model reads sub-agent descriptions to make routing decisions. Write descriptions as if documenting a function's purpose.

3. **Prefer `output_key` + `{key}` templating** over tool-based state manipulation for simple data flow between sequential agents. It is more readable and less error-prone.

4. **Set `max_iterations` on every `LoopAgent`** as a safety net, even when you expect escalation-based exit. Production systems must have bounded execution.

5. **Use unique `output_key` values** for all agents in a `ParallelAgent` to prevent race conditions on shared state.

6. **Keep `global_instruction` brief** (1-2 paragraphs). It is prepended to every LLM call in the tree, so verbosity multiplies token costs.

7. **Choose `AgentTool` vs `sub_agents` deliberately**: `AgentTool` = function call (parent retains control, no conversation history in specialist); `sub_agents` = full transfer (specialist gets full context, parent is out of the loop).

8. **Test agent hierarchies with the ADK Dev UI** (`adk web`). The event graph visualization shows transfer chains, state changes, and tool calls across the entire agent tree, making it significantly easier to debug routing issues.

---

## References

- LLM Agents: https://google.github.io/adk-docs/agents/llm-agents/
- Workflow Agents overview: https://google.github.io/adk-docs/agents/workflow-agents/
- Sequential Agents: https://google.github.io/adk-docs/agents/workflow-agents/sequential-agents/
- Parallel Agents: https://google.github.io/adk-docs/agents/workflow-agents/parallel-agents/
- Loop Agents: https://google.github.io/adk-docs/agents/workflow-agents/loop-agents/
- Custom Agents: https://google.github.io/adk-docs/agents/custom-agents/
- Multi-Agent Systems: https://google.github.io/adk-docs/agents/multi-agents/
- Agent Types overview: https://google.github.io/adk-docs/agents/
- API Reference (Python): https://google.github.io/adk-docs/api-reference/python/google-adk.html
- AgentConfig Reference: https://google.github.io/adk-docs/api-reference/agentconfig/
