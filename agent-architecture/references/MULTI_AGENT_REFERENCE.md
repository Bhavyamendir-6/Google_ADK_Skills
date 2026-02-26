# Multi-Agent Systems — Detailed Reference

## Overview

A Multi-Agent System (MAS) in ADK is an application where multiple `BaseAgent` instances form a hierarchy and collaborate to achieve a larger goal. Benefits include modularity, specialization, reusability, and maintainability.

## Agent Hierarchy

### Setting Up Parent–Child Relationships

```python
from google.adk.agents import LlmAgent

# Define specialist agents
billing_agent = LlmAgent(
    name="billing_agent",
    model="gemini-2.5-flash",
    description="Handles billing inquiries, invoices, and payment issues.",
    instruction="You handle billing-related queries...",
)

shipping_agent = LlmAgent(
    name="shipping_agent",
    model="gemini-2.5-flash",
    description="Handles shipping status, tracking, and delivery issues.",
    instruction="You handle shipping-related queries...",
)

# Create coordinator with sub-agents
coordinator = LlmAgent(
    name="coordinator",
    model="gemini-2.5-flash",
    description="Routes customer requests to the appropriate specialist.",
    instruction="""You are the main coordinator.
Route billing questions to billing_agent.
Route shipping questions to shipping_agent.
If unsure, ask the user to clarify.""",
    sub_agents=[billing_agent, shipping_agent],
)

# ADK automatically sets:
# billing_agent.parent_agent == coordinator
# shipping_agent.parent_agent == coordinator
```

### Rules

- **Single parent:** An agent instance can only belong to one parent.
- **Name uniqueness:** All agents in a hierarchy must have unique names.
- **Navigation:** Use `agent.parent_agent` (up) or `agent.find_agent("name")` (search descendants).

## Interaction Patterns

### 1. LLM-Driven Transfer (Dynamic Delegation)

The LLM decides which sub-agent to route to by generating `transfer_to_agent(agent_name='...')`.

```python
# The coordinator's LLM sees the sub-agents' descriptions and decides:
# User says: "I need to check my invoice"
# LLM generates: transfer_to_agent(agent_name='billing_agent')
# ADK routes execution to billing_agent automatically
```

**Requirements:**
- Coordinator needs clear `instruction` on when to transfer.
- Sub-agents need distinct, specific `description` fields.

### 2. AgentTool (Explicit Invocation)

Wrap a target agent as a tool — the parent LLM calls it like any other function.

```python
from google.adk.agents import LlmAgent
from google.adk.tools.agent_tool import AgentTool

# Define a specialist agent
summarizer = LlmAgent(
    name="summarizer",
    model="gemini-2.5-flash",
    instruction="Summarize the provided text into 3 bullet points.",
)

# Wrap it as a tool
summarizer_tool = AgentTool(agent=summarizer)

# Parent agent uses it as a tool
research_agent = LlmAgent(
    name="research_agent",
    model="gemini-2.5-flash",
    instruction="""You are a research assistant.
When you gather information, use the summarizer tool to create concise summaries.""",
    tools=[summarizer_tool],
)
```

**Key difference from transfer:** AgentTool runs the target agent, captures its response, and returns it as a tool result — the parent agent stays in control.

### 3. Workflow Orchestration (Deterministic Flow)

Use Workflow Agents for structured pipelines (see [Workflow Agent Reference](WORKFLOW_AGENT_REFERENCE.md)).

## Data Passing via State

Agents share data through `session.state` using `output_key`:

```python
from google.adk.agents import LlmAgent, SequentialAgent

# Agent A writes to state
fetcher = LlmAgent(
    name="data_fetcher",
    model="gemini-2.5-flash",
    instruction="Find the population of the requested city.",
    output_key="city_population",
)

# Agent B reads from state via template
reporter = LlmAgent(
    name="reporter",
    model="gemini-2.5-flash",
    instruction="Write a brief report about a city with population: {city_population}.",
    output_key="final_report",
)

pipeline = SequentialAgent(
    name="city_report_pipeline",
    sub_agents=[fetcher, reporter],
)
```

## Common Multi-Agent Patterns

### Pattern 1: Router (Fan-Out)

```python
faq_agent = LlmAgent(name="faq_agent", description="Answers frequently asked questions.", ...)
escalation_agent = LlmAgent(name="escalation_agent", description="Handles complex issues requiring human review.", ...)

router = LlmAgent(
    name="router",
    model="gemini-2.5-flash",
    instruction="Route simple questions to faq_agent. Route complex or angry customer issues to escalation_agent.",
    sub_agents=[faq_agent, escalation_agent],
)
```

### Pattern 2: Parallel Gather + Synthesize

```python
from google.adk.agents import ParallelAgent, SequentialAgent, LlmAgent

api_fetcher = LlmAgent(name="api_fetcher", output_key="api_data", ...)
db_fetcher = LlmAgent(name="db_fetcher", output_key="db_data", ...)

gatherer = ParallelAgent(name="gatherer", sub_agents=[api_fetcher, db_fetcher])

synthesizer = LlmAgent(
    name="synthesizer",
    instruction="Combine the API data ({api_data}) and DB data ({db_data}) into a unified report.",
)

pipeline = SequentialAgent(name="full_pipeline", sub_agents=[gatherer, synthesizer])
```

### Pattern 3: Generator–Critic Loop

```python
from google.adk.agents import LoopAgent, SequentialAgent, LlmAgent

generator = LlmAgent(
    name="draft_writer",
    instruction="Write or revise a draft based on requirements: {requirements}. Previous feedback: {feedback?}.",
    output_key="current_draft",
)

critic = LlmAgent(
    name="reviewer",
    instruction="Review the draft: {current_draft}. If acceptable, respond with exactly 'APPROVED'. Otherwise provide specific feedback.",
    output_key="feedback",
)

review_cycle = SequentialAgent(name="review_cycle", sub_agents=[generator, critic])

iterative_writer = LoopAgent(
    name="iterative_writer",
    sub_agents=[review_cycle],
    max_iterations=5,
)
```

### Pattern 4: Hierarchical Delegation

```python
# Leaf-level tool agents
search_agent = LlmAgent(name="searcher", tools=[web_search], output_key="search_results", ...)
analyzer = LlmAgent(name="analyzer", instruction="Analyze: {search_results}", output_key="analysis", ...)

# Mid-level pipeline
research_team = SequentialAgent(name="research_team", sub_agents=[search_agent, analyzer])

# Top-level coordinator
director = LlmAgent(
    name="director",
    model="gemini-2.5-flash",
    instruction="Delegate research tasks to the research_team.",
    sub_agents=[research_team],
)
```

## Guidelines

1. **Keep agent responsibilities narrow** — each agent should do one thing well.
2. **Write distinct descriptions** — LLM transfer depends on clear differentiation.
3. **Use `output_key` consistently** — it's the primary data passing mechanism.
4. **Avoid state key collisions** in `ParallelAgent` — use unique keys per sub-agent.
5. **Prefer Workflow Agents over Custom Agents** when standard patterns (sequence/parallel/loop) suffice.
