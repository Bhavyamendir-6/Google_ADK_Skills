# Memory (Long-Term Knowledge) — Detailed Reference

## Overview

`MemoryService` provides a searchable, cross-session knowledge store. It allows agents to recall information from past conversations, enabling continuity across multiple sessions.

## Setup

### InMemoryMemoryService (Development)

```python
from google.adk.memory import InMemoryMemoryService

memory_service = InMemoryMemoryService()
# No setup required — data lost on restart
```

### VertexAiMemoryBankService (Production)

```python
from google.adk.memory import VertexAiMemoryBankService

memory_service = VertexAiMemoryBankService(
    project="my-gcp-project",
    location="us-central1",
    agent_engine_id="my-agent-engine-id",
)
# Requires: Google Cloud project, Vertex AI API enabled, Agent Engine instance
```

## Memory Workflow

### Step 1: Ingest Sessions into Memory

After a conversation completes, add it to the memory store:

```python
# Get the completed session
completed_session = await session_service.get_session(
    app_name="my_app", user_id="user_1", session_id="session_abc"
)

# Add to memory
await memory_service.add_session_to_memory(completed_session)
```

### Step 2: Give the Agent a Memory Tool

ADK provides two built-in tools:

**`LoadMemory` — On-demand (agent decides when to search):**

```python
from google.adk.agents import Agent
from google.adk.tools import load_memory

agent = Agent(
    model="gemini-2.5-flash",
    name="assistant",
    instruction="""You are a helpful assistant.
Use the load_memory tool when the user asks about something
from a previous conversation.""",
    tools=[load_memory],
)
```

**`PreloadMemoryTool` — Automatic (loads memory every turn):**

```python
from google.adk.agents import Agent
from google.adk.tools.preload_memory_tool import PreloadMemoryTool

agent = Agent(
    model="gemini-2.5-flash",
    name="assistant",
    instruction="Answer the user's questions using all available context.",
    tools=[PreloadMemoryTool()],
)
```

### Step 3: Wire It Together with a Runner

```python
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.memory import InMemoryMemoryService

session_service = InMemorySessionService()
memory_service = InMemoryMemoryService()

runner = Runner(
    agent=my_agent,
    app_name="my_app",
    session_service=session_service,
    memory_service=memory_service,  # Provide memory service to the runner
)
```

## Full Example: Cross-Session Recall

```python
import asyncio
from google.adk.agents import LlmAgent
from google.adk.sessions import InMemorySessionService
from google.adk.memory import InMemoryMemoryService
from google.adk.runners import Runner
from google.adk.tools import load_memory
from google.genai.types import Content, Part

# --- Services (shared across all runners) ---
session_service = InMemorySessionService()
memory_service = InMemoryMemoryService()

APP_NAME = "travel_assistant"
USER_ID = "traveler_1"

# --- Agents ---
trip_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="trip_agent",
    instruction="Acknowledge the user's travel plans and preferences.",
)

recall_agent = LlmAgent(
    model="gemini-2.5-flash",
    name="recall_agent",
    instruction="""Answer the user's question. If the answer might be
in past conversations, use the load_memory tool to search.""",
    tools=[load_memory],
)


async def demo():
    # --- Session 1: User shares travel preferences ---
    runner1 = Runner(
        agent=trip_agent,
        app_name=APP_NAME,
        session_service=session_service,
        memory_service=memory_service,
    )
    await session_service.create_session(
        app_name=APP_NAME, user_id=USER_ID, session_id="session_preferences"
    )

    msg1 = Content(parts=[Part(text="I love beach destinations. My favorite was Bali.")], role="user")
    async for event in runner1.run_async(
        user_id=USER_ID, session_id="session_preferences", new_message=msg1
    ):
        if event.is_final_response() and event.content:
            print(f"Trip agent: {event.content.parts[0].text}")

    # Ingest session into memory
    session1 = await session_service.get_session(
        app_name=APP_NAME, user_id=USER_ID, session_id="session_preferences"
    )
    await memory_service.add_session_to_memory(session1)
    print("✅ Session ingested into memory")

    # --- Session 2: New conversation, agent recalls from memory ---
    runner2 = Runner(
        agent=recall_agent,
        app_name=APP_NAME,
        session_service=session_service,
        memory_service=memory_service,
    )
    await session_service.create_session(
        app_name=APP_NAME, user_id=USER_ID, session_id="session_planning"
    )

    msg2 = Content(parts=[Part(text="Where should I go for my next vacation?")], role="user")
    async for event in runner2.run_async(
        user_id=USER_ID, session_id="session_planning", new_message=msg2
    ):
        if event.is_final_response() and event.content:
            print(f"Recall agent: {event.content.parts[0].text}")
            # Agent should reference Bali / beach preferences from memory


# asyncio.run(demo())
```

## Automatic Memory Ingestion via Callback

Instead of manually calling `add_session_to_memory`, automate it with an after-agent callback:

```python
from google.adk.agents import Agent
from google.adk.tools.preload_memory_tool import PreloadMemoryTool

async def auto_save_memory(callback_context):
    """After-agent callback that saves session to memory automatically."""
    await callback_context._invocation_context.memory_service.add_session_to_memory(
        callback_context._invocation_context.session
    )

agent = Agent(
    model="gemini-2.5-flash",
    name="auto_memory_agent",
    instruction="Answer the user's questions. Past context is automatically available.",
    tools=[PreloadMemoryTool()],
    after_agent_callback=auto_save_memory,
)
```

## Searching Memory in Custom Tools

```python
async def search_past_context(query: str, tool_context) -> str:
    """Search past conversations for relevant information."""
    results = await tool_context.search_memory(query)

    if not results.memories:
        return "No relevant past context found."

    summaries = []
    for memory in results.memories:
        if memory.content and memory.content.parts:
            summaries.append(memory.content.parts[0].text)

    return "\n".join(summaries)
```

## Key Guidelines

1. **Services must be shared** — use the same `session_service` and `memory_service` across runners
2. **Ingest after completion** — call `add_session_to_memory()` when a session has meaningful content
3. **Choose the right tool** — `PreloadMemoryTool` for always-on recall, `load_memory` for on-demand
4. **InMemory is for dev only** — data is lost on restart; use `VertexAiMemoryBankService` for production
5. **Memory ≠ State** — State is for current session data; Memory is for cross-session knowledge
