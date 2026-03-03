---
name: agent-architecture
description: >
  Covers the structural design of single-agent and multi-agent systems in Google ADK.
  Explains how LlmAgent, workflow agents, and BaseAgent compose into executable pipelines.
  Defines how state, delegation, and sub-agent relationships are wired at the architecture level.
metadata:
  author: adk-skills
  version: "1.0"
---

# Agent Architecture — Detailed Reference

## Overview

Agent Architecture in Google ADK defines how individual agents are constructed, composed, and orchestrated into coherent execution systems. It covers the agent class hierarchy, the parent-child delegation model, state sharing via `session.state`, and the layered execution model from root agent down to leaf agents and tools.

ADK agents are Python objects that inherit from either `BaseAgent` (for custom logic) or one of the built-in agent types: `LlmAgent`, `SequentialAgent`, `ParallelAgent`, or `LoopAgent`. Every runtime invocation starts at a root agent. That root agent may delegate work to sub-agents, which may further delegate, forming a tree.

This skill applies when:
- Designing any multi-agent pipeline in ADK
- Choosing the correct agent class for a task type
- Wiring state between agents using `output_key` and `session.state`
- Structuring delegation and preventing ambiguous routing

---

## Core Concept

### Agent Class Hierarchy

ADK provides a strict class hierarchy:

```
BaseAgent
├── LlmAgent          # LLM-backed reasoning and tool use
├── SequentialAgent   # Deterministic serial execution of sub-agents
├── ParallelAgent     # Concurrent execution of sub-agents
└── LoopAgent         # Repeated execution until termination condition
```

All agent types are composable. Any agent can be placed as a sub-agent of another agent, with the exception that `LlmAgent` sub-agents used as tools must be wrapped with `AgentTool`.

### The Root Agent

Every ADK application has exactly one root agent. This is the entry point passed to the ADK runner. The root agent owns the top-level session context and is responsible for either completing the task directly or delegating to sub-agents.

```python
from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService

root_agent = LlmAgent(
    name="root",
    model="gemini-2.0-flash",
    instruction="You are the orchestrator. Route user requests to the correct sub-agent.",
    sub_agents=[billing_agent, support_agent]
)

session_service = InMemorySessionService()
runner = Runner(agent=root_agent, app_name="my_app", session_service=session_service)
```

### Parent-Child Relationship

Sub-agents are declared at construction time via the `sub_agents` parameter. ADK sets `parent_agent` on each sub-agent automatically at runtime. This relationship is immutable during execution.

- A sub-agent can access its parent via `context.agent.parent_agent`.
- Sub-agents cannot be dynamically added after the agent tree is initialized.
- Each agent in the tree must have a unique `name`.

### State Architecture

Session state (`session.state`) is a shared, mutable dictionary accessible to all agents within a single session. It is the primary mechanism for passing data between agents in a pipeline.

`LlmAgent` uses `output_key` to write its final response text into `session.state` automatically:

```python
summarizer = LlmAgent(
    name="summarizer",
    model="gemini-2.0-flash",
    instruction="Summarize the following text: {raw_text}",
    output_key="summary"
)
```

After `summarizer` runs, `session.state["summary"]` holds the output. Downstream agents reference it via `{summary}` in their instruction templates.

---

## Execution Layers

ADK agent execution follows a layered model:

```
Runner
└── Root Agent (entry point)
    ├── Orchestration Layer  (SequentialAgent / ParallelAgent / LoopAgent)
    │   └── Task Layer       (LlmAgent with tools)
    │       └── Tool Layer   (FunctionTool / AgentTool / built-in tools)
    └── Direct Completion    (LlmAgent handles request without delegation)
```

**Layer responsibilities:**

| Layer | Responsibility |
|---|---|
| Runner | Session lifecycle, event streaming, agent invocation |
| Root Agent | Entry point; delegates or responds directly |
| Orchestration Layer | Controls execution order, concurrency, or iteration |
| Task Layer | LLM reasoning, tool invocation, state writing |
| Tool Layer | External calls, computations, sub-agent invocations |

---

## When to Use Each Agent Type

| Need | Recommended Agent | Why |
|---|---|---|
| LLM reasoning, tool use, natural language response | `LlmAgent` | Backed by Gemini; supports tools and instructions |
| Fixed sequence of steps, each dependent on prior output | `SequentialAgent` | Executes sub-agents in declaration order |
| Independent parallel tasks with no data dependency | `ParallelAgent` | Runs sub-agents concurrently; state writes must not collide |
| Retry, polling, or iterative refinement loops | `LoopAgent` | Repeats sub-agent until escalation or condition met |
| Custom Python logic, non-LLM agents | `BaseAgent` | Full control; implement `_run_async_impl` |
| Exposing an agent as a callable tool to an LlmAgent | `AgentTool` | Wraps an agent for use in `tools` list of another `LlmAgent` |

---

## Common Patterns

### Pattern 1: Hub-and-Spoke Orchestration

A root `LlmAgent` acts as a router. Specialist sub-agents handle discrete domains. The root uses `transfer_to_agent` or LLM-driven delegation to route.

```python
from google.adk.agents import LlmAgent

billing_agent = LlmAgent(
    name="billing_agent",
    model="gemini-2.0-flash",
    instruction="Handle all billing and payment queries.",
    output_key="billing_response"
)

support_agent = LlmAgent(
    name="support_agent",
    model="gemini-2.0-flash",
    instruction="Handle technical support issues.",
    output_key="support_response"
)

root_agent = LlmAgent(
    name="root_agent",
    model="gemini-2.0-flash",
    instruction=(
        "You are a routing agent. "
        "Delegate billing questions to billing_agent. "
        "Delegate technical questions to support_agent."
    ),
    sub_agents=[billing_agent, support_agent]
)
```

### Pattern 2: Sequential Pipeline with State Passing

A `SequentialAgent` chains agents where each step consumes the output of the previous step via `session.state`.

```python
from google.adk.agents import LlmAgent, SequentialAgent

extractor = LlmAgent(
    name="extractor",
    model="gemini-2.0-flash",
    instruction="Extract key facts from: {user_input}",
    output_key="extracted_facts"
)

analyzer = LlmAgent(
    name="analyzer",
    model="gemini-2.0-flash",
    instruction="Analyze these facts and identify risks: {extracted_facts}",
    output_key="risk_analysis"
)

reporter = LlmAgent(
    name="reporter",
    model="gemini-2.0-flash",
    instruction="Write a formal report based on: {risk_analysis}",
    output_key="final_report"
)

pipeline = SequentialAgent(
    name="analysis_pipeline",
    sub_agents=[extractor, analyzer, reporter]
)
```

### Pattern 3: Parallel Fan-Out with Aggregation

Use `ParallelAgent` for independent tasks, then aggregate results with a downstream `LlmAgent`.

```python
from google.adk.agents import LlmAgent, SequentialAgent, ParallelAgent

sentiment_agent = LlmAgent(
    name="sentiment_agent",
    model="gemini-2.0-flash",
    instruction="Analyze sentiment of: {review_text}",
    output_key="sentiment_result"
)

keyword_agent = LlmAgent(
    name="keyword_agent",
    model="gemini-2.0-flash",
    instruction="Extract keywords from: {review_text}",
    output_key="keyword_result"
)

parallel_analysis = ParallelAgent(
    name="parallel_analysis",
    sub_agents=[sentiment_agent, keyword_agent]
)

aggregator = LlmAgent(
    name="aggregator",
    model="gemini-2.0-flash",
    instruction=(
        "Combine these results into a summary:\n"
        "Sentiment: {sentiment_result}\n"
        "Keywords: {keyword_result}"
    ),
    output_key="combined_summary"
)

full_pipeline = SequentialAgent(
    name="review_pipeline",
    sub_agents=[parallel_analysis, aggregator]
)
```

### Pattern 4: Iterative Refinement with LoopAgent

Use `LoopAgent` when an agent must repeat until a quality threshold or external condition is met. Termination is signalled by the sub-agent escalating.

```python
from google.adk.agents import LlmAgent, LoopAgent

refinement_agent = LlmAgent(
    name="refinement_agent",
    model="gemini-2.0-flash",
    instruction=(
        "Review the current draft: {draft}. "
        "If quality is acceptable, set output_key to 'approved' and escalate. "
        "Otherwise rewrite and store improved draft."
    ),
    output_key="draft"
)

refinement_loop = LoopAgent(
    name="refinement_loop",
    sub_agents=[refinement_agent],
    max_iterations=5
)
```

### Pattern 5: Agent-as-Tool

Expose a specialized `LlmAgent` as a tool callable by another `LlmAgent` using `AgentTool`.

```python
from google.adk.agents import LlmAgent, AgentTool

translation_agent = LlmAgent(
    name="translation_agent",
    model="gemini-2.0-flash",
    instruction="Translate the provided text to {target_language}.",
    output_key="translated_text"
)

primary_agent = LlmAgent(
    name="primary_agent",
    model="gemini-2.0-flash",
    instruction="Answer user questions. Use the translation tool when translation is needed.",
    tools=[AgentTool(agent=translation_agent)]
)
```

---

## Rules

- Every agent in a tree must have a unique `name`. Duplicate names cause routing failures.
- `sub_agents` is set at construction time and cannot be mutated at runtime.
- `output_key` writes to `session.state` using the key as the dictionary key. If two agents write to the same key, the last write wins and prior data is silently overwritten.
- `ParallelAgent` sub-agents must write to distinct `output_key` values. Concurrent writes to the same key produce non-deterministic state.
- `LlmAgent` sub-agents referenced via `transfer_to_agent` must be present in the `sub_agents` list of the delegating agent or accessible in the agent tree.
- `AgentTool` wraps an agent for tool-call invocation; the wrapped agent must not itself be listed in `sub_agents` of the same parent.
- `LoopAgent` requires either a `max_iterations` bound or a reliable escalation path in the sub-agent. An unbounded loop without escalation will exhaust token limits.

---

## Edge Cases

### Duplicate Agent Names
If two agents share a `name`, `transfer_to_agent` routing becomes ambiguous. ADK does not enforce uniqueness at construction in all versions; failures appear at runtime as incorrect delegation. Always enforce unique names programmatically.

### State Key Collisions in ParallelAgent
Two sub-agents inside a `ParallelAgent` writing to the same `output_key` will produce a race condition. The resulting state value is non-deterministic. Assign explicit, distinct `output_key` values to every parallel sub-agent.

### Missing output_key in Sequential Pipelines
If an upstream agent in a `SequentialAgent` does not set `output_key`, its output is not persisted to `session.state`. Downstream agents referencing `{that_key}` in their instruction templates will receive an empty or unresolved value. Every intermediate agent in a pipeline must declare `output_key`.

### Circular Delegation
If Agent A delegates to Agent B and Agent B's instruction causes it to delegate back to Agent A, an infinite delegation loop occurs. ADK does not detect circular delegation at construction time. Design instruction prompts to prevent re-delegation to the originating agent.

### AgentTool in sub_agents
Placing an `AgentTool`-wrapped agent also in `sub_agents` of the same parent creates a dual registration. The agent may be invoked twice or exhibit inconsistent behavior. Use `AgentTool` exclusively in the `tools` list, never in `sub_agents`.

### Unbounded LoopAgent
A `LoopAgent` without `max_iterations` and with a sub-agent that never escalates will loop until the session token budget is exhausted. Always set `max_iterations` as a hard upper bound even when relying on escalation for normal termination.

### State Scope Misconception
`session.state` is session-scoped, not agent-scoped. All agents in the same session read and write to the same dictionary. Agents must use namespaced keys (e.g., `"billing.invoice_id"` vs `"support.ticket_id"`) to avoid cross-domain collisions in complex multi-agent systems.

---

## Execution Flow

A typical multi-agent execution flow in ADK:

```
1. Runner.run_async(user_message) called
2. Root agent receives invocation context
3. Root LlmAgent constructs prompt from instruction + session.state values
4. LLM response either:
   a. Contains a final answer → Runner emits completion event
   b. Contains a tool call → ADK executes tool, result appended to context, LLM re-invoked
   c. Contains transfer_to_agent → ADK delegates to named sub-agent, repeats from step 3
5. Sub-agent executes, writes output_key to session.state
6. Control returns to parent agent (if delegation) or Runner (if root completed)
7. Runner streams events to caller
```

---

## Failure Handling Considerations

- **LLM failure (API error):** ADK propagates exceptions from the model call. Implement retry logic at the `Runner` level or wrap agent invocations in try/except within custom `BaseAgent` implementations.
- **Tool execution failure:** A failed tool call returns an error result to the LLM context. The LLM may retry, hallucinate an alternative, or produce a degraded response. Validate tool outputs before writing to `session.state`.
- **Sub-agent not found:** If `transfer_to_agent` names an agent not in the tree, ADK raises a runtime error. Validate agent names in tests before deployment.
- **State deserialization error:** If `session.state` values are expected to be structured (e.g., JSON) but a prior agent wrote malformed data, downstream agents will silently use broken input. Validate state values at agent boundaries.

---

## Scalability Notes

- `ParallelAgent` concurrency is bounded by the underlying async event loop and API rate limits. For large fan-outs (10+ parallel agents), implement rate-limiting at the tool or session level.
- `session.state` is held in memory by `InMemorySessionService`. For production deployments with persistence requirements, replace with a database-backed session service.
- Agent trees with more than 3–4 levels of nesting become difficult to debug. Prefer flat or two-level hierarchies with clear domain boundaries.
- Avoid placing heavy computation inside `LlmAgent` instructions. Offload processing to `FunctionTool` implementations to keep LLM calls focused on reasoning.

---

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
|---|---|---|
| All logic in a single monolithic `LlmAgent` | Unmanageable instructions; poor separation of concerns | Decompose into specialist agents in a `SequentialAgent` or hub-and-spoke |
| Writing all state to a single generic key like `"result"` | Downstream agents overwrite each other's outputs | Use domain-namespaced keys per agent |
| Using `ParallelAgent` for dependent tasks | Race conditions; undefined execution order | Use `SequentialAgent` when tasks depend on prior output |
| Nesting `LoopAgent` inside `LoopAgent` | Exponential iteration counts; hard to bound | Use a single loop with compound sub-agent logic |
| Putting business logic in `instruction` strings | Hard to test; fragile prompt engineering | Move deterministic logic to `FunctionTool`; keep instructions high-level |
| Skipping `output_key` on intermediate agents | Silent data loss between pipeline stages | Every agent in a pipeline must declare an explicit `output_key` |

---

## Guidelines

1. **Assign unique, descriptive names to every agent.** Names are used for routing, debugging, and event attribution. Use snake_case domain-prefixed names (e.g., `billing_invoice_agent`).

2. **Declare `output_key` on every agent that produces data consumed downstream.** Do not rely on the LLM response text being available implicitly; only `session.state` persists across agent boundaries.

3. **Use `SequentialAgent` as the default composition primitive.** Reach for `ParallelAgent` only when tasks are provably independent. Reach for `LoopAgent` only when iteration count is bounded.

4. **Namespace all `session.state` keys in multi-domain systems.** Prefix keys with the owning agent's domain (e.g., `"compliance.check_result"`) to prevent collisions between agents developed independently.

5. **Wrap agents exposed as tools with `AgentTool`, never list them in both `tools` and `sub_agents`.** Dual registration produces undefined invocation behavior.

6. **Always set `max_iterations` on `LoopAgent`.** Even when escalation is the expected termination path, a hard iteration bound prevents runaway loops due to LLM instruction drift.

7. **Validate the agent tree structure in tests before deployment.** Write a test that constructs the full agent tree and asserts all `sub_agents` names are unique, all `output_key` values are non-overlapping where required, and all `transfer_to_agent` targets exist in the tree.

8. **Keep `LlmAgent` instruction templates focused on reasoning, not data transformation.** Delegate data formatting, parsing, and validation to `FunctionTool` implementations. This keeps prompts stable and improves testability.
