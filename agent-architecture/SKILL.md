---
name: agent-architecture
description: >
  Design and implement agent systems using Google ADK. Covers choosing the right
  agent type (LLM, Workflow, Custom), composing multi-agent hierarchies, defining
  agent identity and instructions, equipping agents with tools, and orchestrating
  execution flows. Use when building, refactoring, or reviewing any ADK agent
  application architecture.
metadata:
  author: adk-skills
  version: "1.0"
---

# Agent Architecture with Google ADK

## When to use this skill

Activate this skill when the task involves:

- Designing a new agent or multi-agent system with Google ADK
- Choosing between LLM Agent, Workflow Agent, or Custom Agent
- Structuring parent/sub-agent hierarchies
- Setting up agent delegation (transfer or AgentTool)
- Writing agent instructions, tools, or orchestration logic
- Reviewing or refactoring an existing ADK agent architecture

## Core Concepts

### Agent Types — Decision Guide

| Need | Agent Type | Key Class |
|------|-----------|-----------|
| Natural language reasoning, dynamic tool use, flexible decisions | **LLM Agent** | `LlmAgent` / `Agent` |
| Deterministic execution order (sequence, parallel, loop) | **Workflow Agent** | `SequentialAgent`, `ParallelAgent`, `LoopAgent` |
| Custom conditional logic, external integrations, unique patterns | **Custom Agent** | Subclass of `BaseAgent` |

**Rule of thumb:** Use Workflow Agents for the control flow, LLM Agents for the intelligent tasks, and Custom Agents only when neither fits.

### Agent Identity (LLM Agent)

Every `LlmAgent` requires:

1. **`name`** — Unique string identifier (e.g. `order_support_agent`). Avoid the reserved name `user`.
2. **`model`** — LLM model string (e.g. `gemini-2.5-flash`).
3. **`description`** — Concise capability summary. Critical for multi-agent routing — make it specific enough to differentiate from sibling agents.
4. **`instruction`** — The most important parameter. Define persona, task, constraints, tool usage guidance, and output format. Supports `{state_var}` template syntax.

### Instructions — Best Practices

- Be clear and specific; avoid ambiguity.
- Use Markdown for structure (headings, lists).
- Provide few-shot examples for complex tasks.
- Explain **when** and **why** to use each tool, not just list them.
- Use `{var}` to inject state values; append `?` for optional vars (`{var?}`).

### Tools

Tools extend agent capabilities beyond the LLM's built-in knowledge:

- **Function tools** — Native functions wrapped automatically (Python) or via `FunctionTool` (Java/TS).
- **BaseTool subclasses** — Custom tool implementations.
- **AgentTool** — Wraps another agent as a callable tool.

The LLM decides which tool to call based on function name, description (docstring), and parameter schema.

## Multi-Agent Architecture

### Hierarchy

- Assign sub-agents via the `sub_agents` parameter.
- ADK auto-sets `parent_agent` on each child — single parent rule.
- Navigate with `agent.parent_agent` or `agent.find_agent(name)`.

### Agent Interaction Patterns

| Pattern | Mechanism | Best For |
|---------|-----------|----------|
| **Workflow orchestration** | `SequentialAgent`, `ParallelAgent`, `LoopAgent` | Deterministic pipelines |
| **LLM-driven delegation** | `transfer_to_agent(agent_name=...)` | Dynamic routing based on context |
| **Explicit invocation** | `AgentTool` wrapping a target agent | Treating an agent as a tool call |

### Data Passing Between Agents

- Use `output_key` on an LLM Agent to save its output to `session.state`.
- Downstream agents read state via `{key}` in their instruction template.
- In parallel execution, use **distinct keys** to avoid race conditions.

## Workflow Agents

### SequentialAgent

Runs sub-agents one after another. Use for **pipelines** where step N depends on step N-1.

### ParallelAgent

Runs sub-agents concurrently. Shares the same `session.state`. Use for **independent data gathering** tasks.

### LoopAgent

Repeatedly runs sub-agents until a termination condition. Use for **iterative refinement** patterns (e.g. generator–critic loops).

## Custom Agents

Extend `BaseAgent` and implement `_run_async_impl` (Python) / `runAsyncImpl` (TS/Java) when you need:

- Conditional branching based on runtime state
- Complex state management beyond simple sequential passing
- Direct external API/database calls in orchestration logic
- Dynamic agent selection

## Common Architecture Patterns

1. **Router pattern** — A coordinator LLM Agent with sub-agents, using LLM transfer to route user requests to the right specialist.
2. **Sequential pipeline** — `SequentialAgent` chaining fetch → process → report agents with state passing via `output_key`.
3. **Parallel gather + synthesize** — `ParallelAgent` for concurrent data collection, followed by a synthesizer agent in a `SequentialAgent` wrapper.
4. **Generator–critic loop** — `LoopAgent` wrapping a generator agent and a quality-checker agent that stops the loop on "pass".
5. **Hierarchical delegation** — Multi-level tree with a top-level coordinator delegating to mid-level agents, which further delegate to leaf-level tool agents.

## References

- See [LLM Agent reference](references/LLM_AGENT_REFERENCE.md) for detailed configuration, code examples, and advanced features.
- See [Multi-Agent reference](references/MULTI_AGENT_REFERENCE.md) for hierarchy setup, interaction patterns, and architecture examples with code.
- See [Workflow Agent reference](references/WORKFLOW_AGENT_REFERENCE.md) for Sequential, Parallel, and Loop agent details.

## Edge Cases

- **Ambiguous agent type:** Default to LLM Agent; only switch to Workflow or Custom when deterministic control is explicitly needed.
- **Agent name collisions:** All agent names within a hierarchy must be unique — ADK uses `find_agent(name)` for lookups.
- **Circular delegation:** Avoid transfer loops between agents; ensure clear routing rules in instructions.
- **State key conflicts in parallel:** When using `ParallelAgent`, ensure each sub-agent writes to a unique `output_key`.
- **Overly long instructions:** If instructions exceed ~2000 tokens, extract examples and reference material into `references/`.
