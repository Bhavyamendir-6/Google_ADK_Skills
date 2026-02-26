# Workflow Agents — Detailed Reference

## Overview

Workflow Agents are specialized agents that orchestrate the execution flow of sub-agents **without using an LLM for flow control**. They provide deterministic, predictable execution patterns.

## SequentialAgent

Executes sub-agents **one after another** in listed order. The same `InvocationContext` is passed through, so agents can share data via `session.state`.

### When to Use

- Multi-step pipelines where Step N depends on Step N-1
- ETL-style flows (extract → transform → load)
- Any ordered workflow

### Example: Data Processing Pipeline

```python
from google.adk.agents import SequentialAgent, LlmAgent

extractor = LlmAgent(
    name="extractor",
    model="gemini-2.5-flash",
    instruction="Extract key facts from the user's input text.",
    output_key="raw_facts",
)

transformer = LlmAgent(
    name="transformer",
    model="gemini-2.5-flash",
    instruction="Organize these facts into categories: {raw_facts}.",
    output_key="organized_facts",
)

formatter = LlmAgent(
    name="formatter",
    model="gemini-2.5-flash",
    instruction="Format the organized facts into a clean markdown report: {organized_facts}.",
    output_key="final_report",
)

pipeline = SequentialAgent(
    name="fact_pipeline",
    sub_agents=[extractor, transformer, formatter],
)
```

## ParallelAgent

Executes sub-agents **concurrently**. All children share the same `session.state` but events may be interleaved.

### When to Use

- Independent data gathering from multiple sources
- Tasks that don't depend on each other
- Reducing total latency by running tasks simultaneously

### Rules

- Use **distinct `output_key` values** for each sub-agent to avoid race conditions.
- Despite parallel branches, state is shared — agents can read initial state set before the parallel block.

### Example: Multi-Source Information Gathering

```python
from google.adk.agents import ParallelAgent, SequentialAgent, LlmAgent

weather_agent = LlmAgent(
    name="weather_agent",
    model="gemini-2.5-flash",
    instruction="Get the current weather for the destination city: {destination}.",
    output_key="weather_info",
)

flight_agent = LlmAgent(
    name="flight_agent",
    model="gemini-2.5-flash",
    instruction="Find available flights to {destination} for {travel_date}.",
    output_key="flight_info",
)

hotel_agent = LlmAgent(
    name="hotel_agent",
    model="gemini-2.5-flash",
    instruction="Find available hotels in {destination} for {travel_date}.",
    output_key="hotel_info",
)

# Run all three concurrently
info_gatherer = ParallelAgent(
    name="info_gatherer",
    sub_agents=[weather_agent, flight_agent, hotel_agent],
)

# Then synthesize results
trip_planner = LlmAgent(
    name="trip_planner",
    model="gemini-2.5-flash",
    instruction="""Create a trip summary using:
- Weather: {weather_info}
- Flights: {flight_info}
- Hotels: {hotel_info}""",
)

full_pipeline = SequentialAgent(
    name="trip_planning_pipeline",
    sub_agents=[info_gatherer, trip_planner],
)
```

## LoopAgent

**Repeatedly** executes its sub-agents until a termination condition is met.

### When to Use

- Iterative refinement (write → review → revise)
- Retry-until-success patterns
- Polling or monitoring workflows

### Termination Conditions

- **`max_iterations`** — Maximum number of loop cycles.
- **Agent escalation** — A sub-agent can call `escalate` to break out of the loop.

### Example: Iterative Code Refinement

```python
from google.adk.agents import LoopAgent, SequentialAgent, LlmAgent

coder = LlmAgent(
    name="coder",
    model="gemini-2.5-flash",
    instruction="""Write or refine Python code based on:
Requirements: {requirements}
Previous feedback: {review_feedback?}
Current code: {current_code?}""",
    output_key="current_code",
)

reviewer = LlmAgent(
    name="reviewer",
    model="gemini-2.5-flash",
    instruction="""Review this Python code: {current_code}
Check for: correctness, readability, edge cases.
If the code is production-ready, respond with exactly: LGTM
Otherwise, provide specific improvement suggestions.""",
    output_key="review_feedback",
)

code_cycle = SequentialAgent(
    name="code_cycle",
    sub_agents=[coder, reviewer],
)

refinement_loop = LoopAgent(
    name="refinement_loop",
    sub_agents=[code_cycle],
    max_iterations=3,
)
```

## Combining Workflow Agents

Workflow agents compose naturally — nest them to build complex flows:

```python
from google.adk.agents import SequentialAgent, ParallelAgent, LoopAgent, LlmAgent

# Phase 1: Gather data in parallel
source_a = LlmAgent(name="source_a", output_key="data_a", ...)
source_b = LlmAgent(name="source_b", output_key="data_b", ...)
gather_phase = ParallelAgent(name="gather", sub_agents=[source_a, source_b])

# Phase 2: Process sequentially
processor = LlmAgent(name="processor", instruction="Merge {data_a} and {data_b}.", output_key="merged", ...)
validator = LlmAgent(name="validator", instruction="Validate: {merged}.", output_key="status", ...)
process_phase = SequentialAgent(name="process", sub_agents=[processor, validator])

# Phase 3: Retry if validation fails (using loop)
retry_loop = LoopAgent(name="retry", sub_agents=[process_phase], max_iterations=3)

# Full pipeline: gather → process (with retry)
full_pipeline = SequentialAgent(
    name="full_pipeline",
    sub_agents=[gather_phase, retry_loop],
)
```

## Key Guidelines

1. **Workflow agents don't use LLMs** — they're pure orchestration logic.
2. **Sub-agents can be any type** — LLM, Workflow, or Custom agents nest freely.
3. **State is the data bus** — use `output_key` + `{var}` templates for inter-agent communication.
4. **Keep pipelines shallow** — deeply nested workflows are hard to debug; prefer flat sequential chains.
5. **Use `max_iterations` defensively** — always set a cap on `LoopAgent` to prevent infinite loops.
