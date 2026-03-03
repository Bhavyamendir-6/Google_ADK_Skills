---
name: memory-management
description: >
  Covers all memory and state persistence mechanisms available in Google ADK.
  Explains session.state, MemoryService, SessionService, and artifact storage as distinct layers.
  Defines when and how to use each layer for short-term, long-term, and cross-session data retention.
metadata:
  author: adk-skills
  version: "1.0"
---

# Memory Management — Detailed Reference

## Overview

Memory management in Google ADK is the set of mechanisms by which agents read, write, and persist data across turns, across agents within a session, and across sessions entirely. ADK separates memory into four distinct layers: in-turn context (LLM conversation history), session state (`session.state`), session persistence (`SessionService`), and long-term semantic memory (`MemoryService`). A fifth layer — artifact storage — handles binary and structured file data.

Understanding which layer to use is mandatory for building correct multi-agent pipelines. Using `session.state` as a substitute for long-term memory, or using `MemoryService` for ephemeral turn data, are common architectural errors with silent failure modes.

This skill applies when:
- Passing structured data between agents within a single session
- Persisting user context or preferences across multiple sessions
- Enabling agents to perform semantic search over past interactions
- Storing and retrieving files, documents, or binary artifacts within a pipeline
- Diagnosing state loss or stale data bugs in multi-agent systems

---

## Memory Layer Architecture

ADK organizes memory into five layers with distinct scopes, lifetimes, and access patterns:

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: LLM Conversation History (in-context)             │
│  Scope: single agent invocation turn                        │
│  Managed by: ADK event loop automatically                   │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: session.state (in-session shared state)           │
│  Scope: all agents within one session                       │
│  Managed by: agents via output_key and direct writes        │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: SessionService (session persistence)              │
│  Scope: full session data, survives process restarts        │
│  Managed by: Runner + SessionService implementation         │
├─────────────────────────────────────────────────────────────┤
│  Layer 4: MemoryService (long-term semantic memory)         │
│  Scope: cross-session; queryable by content similarity      │
│  Managed by: agents via MemoryService.add_session_to_memory │
├─────────────────────────────────────────────────────────────┤
│  Layer 5: ArtifactService (file/binary storage)             │
│  Scope: named versioned artifacts; session or user scoped   │
│  Managed by: agents via context.save_artifact / load_artifact│
└─────────────────────────────────────────────────────────────┘
```

---

## Core Concept

### Layer 1 — LLM Conversation History

ADK automatically maintains the conversation history for each `LlmAgent` invocation as a sequence of `Content` events. This history is passed to the model on every turn within the same session. Agents do not manually manage this layer.

- History is scoped to the current session and agent invocation context.
- It is not directly accessible as a Python data structure during normal tool execution.
- It resets when a new session is created.

### Layer 2 — session.state

`session.state` is a plain Python dictionary shared across all agents in a session. It is the primary mechanism for structured data passing between agents.

```python
from google.adk.agents import LlmAgent, SequentialAgent

fetch_agent = LlmAgent(
    name="fetch_agent",
    model="gemini-2.0-flash",
    instruction="Fetch the user's account tier from the system.",
    output_key="account_tier"
)

response_agent = LlmAgent(
    name="response_agent",
    model="gemini-2.0-flash",
    instruction=(
        "The user has account tier: {account_tier}. "
        "Tailor your response accordingly."
    ),
    output_key="final_response"
)

pipeline = SequentialAgent(
    name="account_pipeline",
    sub_agents=[fetch_agent, response_agent]
)
```

Agents can also read and write `session.state` directly inside `FunctionTool` implementations:

```python
from google.adk.tools import FunctionTool

def record_user_preference(topic: str, tool_context) -> str:
    tool_context.state["user_preference"] = topic
    return f"Preference '{topic}' recorded."

preference_tool = FunctionTool(func=record_user_preference)
```

### Layer 3 — SessionService

`SessionService` persists the full session object, including `session.state` and conversation history, so it survives process restarts and can be resumed. ADK provides two built-in implementations:

| Implementation | Use Case |
|---|---|
| `InMemorySessionService` | Development, testing, single-process deployments |
| `DatabaseSessionService` | Production; requires a database backend connection string |

```python
from google.adk.sessions import DatabaseSessionService
from google.adk.runners import Runner
from google.adk.agents import LlmAgent

session_service = DatabaseSessionService(db_url="postgresql://user:pass@host/db")

root_agent = LlmAgent(
    name="root_agent",
    model="gemini-2.0-flash",
    instruction="You are a persistent assistant."
)

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    session_service=session_service
)
```

Sessions are identified by `app_name` + `user_id` + `session_id`. To resume a session, pass the same `session_id` to `runner.run_async`.

```python
import asyncio
from google.adk.sessions import Session

async def resume_session():
    session = await session_service.get_session(
        app_name="my_app",
        user_id="user_123",
        session_id="session_abc"
    )
    # session.state contains previously written values
    print(session.state)
```

### Layer 4 — MemoryService

`MemoryService` provides long-term semantic memory that persists beyond individual sessions and is queryable by content similarity. After a session ends, an agent or application code calls `MemoryService.add_session_to_memory` to ingest the session. Future sessions can query this memory to retrieve relevant past context.

ADK provides:

| Implementation | Use Case |
|---|---|
| `InMemoryMemoryService` | Development and testing only |
| `VertexAiRagMemoryService` | Production; backed by Vertex AI RAG Engine |

```python
from google.adk.memory import VertexAiRagMemoryService
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.adk.agents import LlmAgent

memory_service = VertexAiRagMemoryService(
    rag_corpus="projects/my-project/locations/us-central1/ragCorpora/my-corpus"
)

session_service = InMemorySessionService()

root_agent = LlmAgent(
    name="root_agent",
    model="gemini-2.0-flash",
    instruction=(
        "You have access to memory about past interactions. "
        "Use the load_memory tool to retrieve relevant context before answering."
    )
)

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    session_service=session_service,
    memory_service=memory_service
)
```

After a session completes, ingest it into long-term memory:

```python
async def ingest_completed_session(session_id: str):
    session = await session_service.get_session(
        app_name="my_app",
        user_id="user_123",
        session_id=session_id
    )
    await memory_service.add_session_to_memory(session)
```

When `memory_service` is provided to the `Runner`, ADK automatically makes a `load_memory` tool available to `LlmAgent` instances. The agent can call this tool to perform semantic search over ingested sessions.

### Layer 5 — ArtifactService

Artifacts store named, versioned binary or text blobs (PDFs, images, JSON files) that are too large or unsuitable for `session.state`. Artifacts are saved and loaded within agent execution via the invocation context.

```python
from google.adk.agents import BaseAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.artifacts import InMemoryArtifactService
import google.genai.types as types

class DocumentProcessorAgent(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext):
        # Save a document artifact
        artifact_part = types.Part.from_text("Processed report content here.")
        await ctx.save_artifact(filename="report.txt", artifact=artifact_part)

        # Load an artifact in a subsequent step
        loaded = await ctx.load_artifact(filename="report.txt")
        # loaded is a types.Part
```

Artifacts are registered with the `Runner` via `artifact_service`:

```python
from google.adk.artifacts import InMemoryArtifactService

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    session_service=session_service,
    artifact_service=InMemoryArtifactService()
)
```

---

## When to Use

- **`session.state`**: Pass structured key-value data between agents within the same session. Use for intermediate pipeline outputs, flags, and runtime configuration.
- **`SessionService` (persistent)**: Resume sessions across process restarts or user reconnections. Required for production deployments where session continuity matters.
- **`MemoryService`**: Enable agents to recall information from past sessions semantically. Use for personalization, long-term user profiling, and context retrieval across conversations.
- **`ArtifactService`**: Store files, reports, images, or large structured data that cannot fit in `session.state` values or should be versioned independently.

---

## Decision Table

| Data Type | Scope | Recommended Layer |
|---|---|---|
| Agent pipeline intermediate output | Within session | `session.state` via `output_key` |
| User preferences set in this session | Within session | `session.state` via `FunctionTool` write |
| Full session recovery after crash | Across process restarts | `DatabaseSessionService` |
| Recall what user said 3 sessions ago | Cross-session semantic | `MemoryService` |
| Store a PDF generated during pipeline | Session or user scoped | `ArtifactService` |
| Large JSON response from external API | Within session | `ArtifactService` (not `session.state`) |
| Feature flag or routing decision | Within session | `session.state` direct write |

---

## Common Patterns

### Pattern 1: Output Key Chain

Each agent in a `SequentialAgent` writes its result to `session.state` via `output_key`. Downstream agents read previous outputs via instruction template variables.

```python
step_one = LlmAgent(
    name="step_one",
    model="gemini-2.0-flash",
    instruction="Classify the intent of: {user_input}",
    output_key="intent"
)

step_two = LlmAgent(
    name="step_two",
    model="gemini-2.0-flash",
    instruction="Given intent '{intent}', generate a response plan.",
    output_key="response_plan"
)

pipeline = SequentialAgent(
    name="intent_pipeline",
    sub_agents=[step_one, step_two]
)
```

### Pattern 2: FunctionTool State Write

A `FunctionTool` writes structured data (dicts, lists) to `session.state` that cannot be naturally expressed as an LLM text output.

```python
from google.adk.tools import FunctionTool
from google.adk.agents import LlmAgent

def save_extracted_entities(entities: list, tool_context) -> str:
    tool_context.state["extracted_entities"] = entities
    return f"Saved {len(entities)} entities."

entity_agent = LlmAgent(
    name="entity_agent",
    model="gemini-2.0-flash",
    instruction="Extract all named entities from the user message and save them.",
    tools=[FunctionTool(func=save_extracted_entities)]
)
```

### Pattern 3: Session Resume with Persistent State

Use `DatabaseSessionService` and a fixed `session_id` per user to maintain continuity across interactions.

```python
from google.adk.sessions import DatabaseSessionService
from google.adk.runners import Runner

session_service = DatabaseSessionService(db_url="postgresql://user:pass@host/db")

runner = Runner(
    agent=root_agent,
    app_name="support_app",
    session_service=session_service
)

async def handle_user_message(user_id: str, session_id: str, message: str):
    async for event in runner.run_async(
        user_id=user_id,
        session_id=session_id,
        new_message=message
    ):
        if event.is_final_response():
            return event.content
```

### Pattern 4: Post-Session Memory Ingestion

After each session ends, ingest it into `MemoryService` so future sessions can semantically query it.

```python
from google.adk.memory import VertexAiRagMemoryService

memory_service = VertexAiRagMemoryService(
    rag_corpus="projects/my-project/locations/us-central1/ragCorpora/corpus-id"
)

async def on_session_end(session_id: str, user_id: str):
    session = await session_service.get_session(
        app_name="my_app",
        user_id=user_id,
        session_id=session_id
    )
    await memory_service.add_session_to_memory(session)
```

### Pattern 5: Artifact Pass-Through in Multi-Agent Pipeline

One agent generates a file artifact; a downstream agent loads and processes it.

```python
from google.adk.agents import BaseAgent
from google.adk.agents.invocation_context import InvocationContext
import google.genai.types as types

class ReportGeneratorAgent(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext):
        report_text = "Quarterly analysis complete. Revenue up 12%."
        artifact = types.Part.from_text(report_text)
        await ctx.save_artifact(filename="quarterly_report.txt", artifact=artifact)

class ReportFormatterAgent(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext):
        report = await ctx.load_artifact(filename="quarterly_report.txt")
        formatted = f"FORMATTED:\n{report.text}"
        await ctx.save_artifact(
            filename="formatted_report.txt",
            artifact=types.Part.from_text(formatted)
        )
```

---

## Rules

- `session.state` values must be JSON-serializable when using `DatabaseSessionService`. Non-serializable Python objects (e.g., class instances, file handles) will cause persistence errors.
- `output_key` only writes the LLM's final text response as a string. To write structured data (dicts, lists), use a `FunctionTool` that writes to `tool_context.state` directly.
- `session.state` is shared across all agents in the session. There is no per-agent state namespace enforced by ADK. Agents must self-enforce key namespacing.
- `MemoryService.add_session_to_memory` is not called automatically. Application code is responsible for invoking it after session completion.
- `load_memory` tool is only available to `LlmAgent` instances when `memory_service` is registered with the `Runner`. It is not available to `BaseAgent` subclasses by default.
- Artifact filenames must be unique within a session scope. Saving to the same filename creates a new version; `load_artifact` returns the latest version by default.
- `InMemorySessionService` and `InMemoryMemoryService` do not persist across process restarts. They must not be used in production deployments requiring durability.

---

## Edge Cases

### Non-Serializable session.state Values
Writing a Python object (e.g., a Pydantic model instance, a datetime object) to `session.state` works correctly with `InMemorySessionService` but raises a serialization error when `DatabaseSessionService` attempts to persist the session. Always convert values to JSON-compatible primitives before writing to state.

### output_key Overwrites Structured State
If a `FunctionTool` writes a dict to `session.state["analysis"]` and a subsequent `LlmAgent` also has `output_key="analysis"`, the LLM text output will silently overwrite the structured dict. Use distinct keys for tool-written structured data and LLM text outputs.

### Memory Ingestion Skipped on Crash
If the application process crashes before `add_session_to_memory` is called, the session is permanently lost from long-term memory. Implement ingestion as a durable background job or transactional hook rather than inline post-session logic.

### Stale State from Previous Sessions
When `DatabaseSessionService` resumes a session, all prior `session.state` values are restored. Agents that assume a clean state on first run may behave incorrectly if keys from a previous turn already exist. Agents must check for key existence before writing defaults:

```python
def initialize_state(tool_context) -> str:
    if "run_count" not in tool_context.state:
        tool_context.state["run_count"] = 0
    tool_context.state["run_count"] += 1
    return "State initialized."
```

### ParallelAgent State Write Collisions
Two agents in a `ParallelAgent` writing to the same `session.state` key concurrently produce non-deterministic results. The final value depends on execution order, which is not guaranteed. Assign unique state keys to each parallel branch.

### load_memory Hallucinated Results
`MemoryService` performs semantic similarity search. The retrieved memory chunks may be topically related but contextually incorrect for the current query. Agents must be instructed to treat retrieved memory as supporting context, not ground truth, and should cite uncertainty when relying on it.

### Artifact Version Confusion
Loading an artifact without specifying a version always returns the latest version. If an earlier pipeline stage saved version 1 and a later stage saved version 2, intermediate agents replaying the pipeline will get version 2, not version 1. Use explicit version parameters when pipeline reproducibility is required.

---

## Failure Handling Considerations

- **`SessionService` write failure:** If the database is unavailable when ADK attempts to persist session state, the `Runner` will raise an exception. Wrap `runner.run_async` calls with retry logic and circuit breakers in production.
- **`MemoryService` query timeout:** Vertex AI RAG queries are network calls and can time out. The `load_memory` tool will return an error to the LLM context. Instruct agents to proceed without memory if the tool returns an error rather than retrying indefinitely.
- **Corrupt `session.state`:** If a prior agent wrote malformed data to a key (e.g., a truncated JSON string due to LLM output cutoff), downstream agents reading that key will silently operate on bad data. Validate state values at critical pipeline stages using `FunctionTool` validators.
- **Missing `artifact_service`:** Calling `ctx.save_artifact` or `ctx.load_artifact` without an `ArtifactService` registered on the `Runner` raises a runtime error. Always register `artifact_service` before deploying agents that use artifacts.

---

## Scalability Notes

- `session.state` size is unbounded by ADK but contributes to session storage cost in `DatabaseSessionService`. Keep individual state values under 10 KB; use `ArtifactService` for larger payloads.
- `VertexAiRagMemoryService` scales with Vertex AI RAG Engine quotas. For high-ingestion workloads, batch `add_session_to_memory` calls and implement rate limiting.
- For multi-tenant applications, always scope sessions with distinct `user_id` values. Sharing a `session_id` across users merges their `session.state`, which is a data isolation violation.
- `InMemoryArtifactService` holds all artifacts in process memory. In pipelines that generate large artifacts (e.g., PDF reports), this can exhaust memory. Use a GCS-backed artifact service in production.

---

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
|---|---|---|
| Using `session.state` as long-term memory | State is session-scoped; lost when session ends | Use `MemoryService` for cross-session persistence |
| Writing large blobs to `session.state` | Increases serialization cost; risks exceeding storage limits | Use `ArtifactService` for payloads over 10 KB |
| Relying on `InMemorySessionService` in production | State lost on process restart | Use `DatabaseSessionService` with a durable backend |
| Forgetting to call `add_session_to_memory` | Sessions never enter long-term memory | Automate ingestion via post-session hook or job queue |
| Using the same state key in `ParallelAgent` branches | Non-deterministic state overwrite | Assign unique, branch-specific state keys |
| Assuming `load_memory` returns exact facts | RAG retrieval is approximate; not transactional | Instruct agents to treat memory as context, not source of truth |
| Storing Python objects in `session.state` | Fails serialization with `DatabaseSessionService` | Serialize to JSON-compatible types before writing |

---

## Guidelines

1. **Determine the correct memory layer before writing any agent.** Ask: does this data need to survive the session? If no, use `session.state`. If yes, use `SessionService` for structured data or `MemoryService` for semantic recall.

2. **Namespace all `session.state` keys.** In multi-agent systems, prefix every key with the writing agent's domain (e.g., `"billing.last_invoice"`, `"support.ticket_id"`) to prevent silent overwrites.

3. **Use `FunctionTool` for structured state writes, not `output_key`.** `output_key` only persists a plain string. For dicts, lists, or typed data, always write via `tool_context.state` inside a `FunctionTool`.

4. **Always register `DatabaseSessionService` in production.** `InMemorySessionService` is suitable only for development and automated tests. Any user-facing deployment requires durable session persistence.

5. **Automate `MemoryService` ingestion as a post-session job.** Do not rely on inline post-session code that can be skipped if the process crashes. Use a task queue or database trigger to call `add_session_to_memory` reliably.

6. **Validate `session.state` at pipeline stage boundaries.** Before a critical `LlmAgent` reads from `session.state`, insert a `FunctionTool`-based validation step that asserts the required keys exist and have the expected types.

7. **Use `ArtifactService` for any payload over 10 KB.** Large values in `session.state` increase database write cost and risk exceeding row size limits. Artifacts are versioned, named, and designed for binary payloads.

8. **Scope `MemoryService` queries with user context.** When calling `load_memory`, ensure the query is scoped to the current user's ingested sessions. Unscoped queries may retrieve memory from other users in shared-corpus deployments.
