---
name: context-and-events
description: >
  Comprehensive reference for the context, event, and artifact systems in Google Agent Development
  Kit (ADK) Python. Covers Agent Context objects (`InvocationContext`, `ReadonlyContext`,
  `CallbackContext`, `ToolContext`, context hierarchy and capabilities), Context Caching
  (`ContextCacheConfig`, `static_instruction`, `App`-level caching configuration, `min_tokens`,
  `ttl_seconds`, `cache_intervals`), Context Compression/Compaction (`EventsCompactionConfig`,
  `LlmEventSummarizer`, sliding window summarization, `compaction_interval`, `overlap_size`),
  Events System (`Event`, `EventActions`, `state_delta`, `artifact_delta`, `transfer_to_agent`,
  `escalate`, `skip_summarization`, `is_final_response()`, `get_function_calls()`,
  `get_function_responses()`, event processing in the Runner loop, yield/pause/commit cycle),
  and Artifacts (`BaseArtifactService`, `InMemoryArtifactService`, `GcsArtifactService`,
  `save_artifact`, `load_artifact`, `list_artifacts`, artifact versioning, `artifact_delta`).
  Consult when building agents that need to access runtime information, optimize token usage
  via caching or compression, process events from the Runner, or store/retrieve binary data.
metadata:
  author: adk-skills
  version: "1.0"
---

# Context & Events — Detailed Reference

## Overview

This skill covers the context, event, and artifact systems in Google ADK (Python). These three systems form the runtime backbone through which agents access information, communicate results, and persist binary data:

1. **Agent Context** — the hierarchy of context objects (`InvocationContext`, `ReadonlyContext`, `CallbackContext`, `ToolContext`) that provide agents, callbacks, and tools with access to session state, services, and runtime identifiers.
2. **Context Caching** — `ContextCacheConfig` configured on the `App` object, enabling the Gemini API to cache repeated prompt content across invocations to reduce latency and cost.
3. **Context Compression** — `EventsCompactionConfig` configured on the `App` object, using a sliding window summarization strategy to condense older event history and manage token budgets for long conversations.
4. **Events System** — `Event` and `EventActions` objects that serve as the fundamental communication and state-change mechanism in ADK's yield/pause/commit runtime loop.
5. **Artifacts** — `BaseArtifactService` implementations (`InMemoryArtifactService`, `GcsArtifactService`) that store and retrieve versioned binary data (images, PDFs, files) associated with sessions.

All five systems sit in the **runtime layer** of the ADK architecture. The Runner creates the `InvocationContext`, processes yielded `Event` objects (committing `state_delta` and `artifact_delta`), and manages caching and compaction through the `App` wrapper.

For session state management and state prefixes, see `sessions-state-memory.md`.
For tool-specific context capabilities (authentication, memory search), see `tools-and-integrations.md`.

---

## Prerequisites

- Python 3.10+
- `pip install google-adk`
- For context caching: Gemini 2.0 or higher model; `App` object wrapping the root agent
- For context compression: ADK v1.16.0+ for `EventsCompactionConfig`; `App` object wrapping the root agent
- For GCS artifact storage: `pip install google-cloud-storage`; GCS bucket with write access; `GOOGLE_CLOUD_PROJECT` env var set
- For Vertex AI-specific context features: `GOOGLE_GENAI_USE_VERTEXAI=TRUE`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION` env vars set

---

## Core Concept

### Context Object Hierarchy

ADK provides a layered hierarchy of context objects. Each layer adds capabilities on top of the previous:

```
┌─────────────────────────────────────────────────────────────┐
│                    InvocationContext                         │
│  Full access: session, agent, services, end_invocation      │
│  Used in: BaseAgent._run_async_impl, _run_live_impl         │
├─────────────────────────────────────────────────────────────┤
│                    ReadonlyContext                           │
│  Read-only: invocation_id, agent_name, state (read-only)    │
│  Used in: InstructionProvider functions                      │
├─────────────────────────────────────────────────────────────┤
│                    CallbackContext                           │
│  Adds: mutable state, save_artifact, load_artifact,         │
│        user_content access                                  │
│  Used in: before/after_agent_callback, before/after_model   │
├─────────────────────────────────────────────────────────────┤
│                      ToolContext                             │
│  Adds: request_credential, get_auth_response, list_artifacts│
│        search_memory, function_call_id, actions              │
│  Used in: FunctionTool functions, before/after_tool_callback │
└─────────────────────────────────────────────────────────────┘
```

The framework creates the `InvocationContext` when `runner.run_async()` is called, populating it with the session, user content, and references to all registered services (`SessionService`, `ArtifactService`, `MemoryService`). Callbacks and tools receive their respective specialized context objects automatically.

**Unified `Context` class**: In the current ADK Python API, `CallbackContext` and `ToolContext` have been unified into a single class: `google.adk.agents.Context`. This class exposes all capabilities, but tool-specific methods (`request_credential`, `request_confirmation`) require `function_call_id` to be set — which happens automatically when the framework creates the context for a tool function. For callbacks (where `function_call_id` is `None`), those methods raise `ValueError`. The official docs still describe the conceptual hierarchy (`ReadonlyContext` → `CallbackContext` → `ToolContext`) for explanatory purposes, and both the legacy import paths (`from google.adk.tools import ToolContext`, `from google.adk.agents.callback_context import CallbackContext`) and the unified import (`from google.adk.agents import Context`) work. Callback type signatures in the API reference use `Callable[[google.adk.agents.context.Context], ...]`.

### Event-Driven Communication

Events are the fundamental unit of communication in ADK. Every interaction — user input, LLM responses, tool calls, tool results, state changes, control signals — flows as an `Event` object through the Runner event loop.

```
runner.run_async()
    │
    ▼
┌──────────────┐     yield Event     ┌──────────────┐
│  Agent Logic  │ ──────────────────► │    Runner     │
│  (or Tool,    │                     │               │
│   Callback)   │ ◄─────────────────  │  Commits:     │
│               │   resume execution  │  state_delta   │
└──────────────┘                      │  artifact_delta│
                                      │  append_event  │
                                      └──────────────┘
```

**The yield/pause/commit cycle**: When agent logic yields an `Event`, execution pauses. The Runner processes the event — committing `state_delta` to the `SessionService`, recording `artifact_delta`, and appending the event to session history. Only after the Runner finishes does execution resume. This guarantees that state changes from a yielded event are persisted before subsequent code runs.

### Context Caching and Compression at the App Level

Both context caching and context compression are configured on the `App` object, not on individual agents:

```
┌──────────────────────────────────────────────────────────┐
│                         App                               │
│                                                           │
│  root_agent ──────────────► Agent tree                    │
│                                                           │
│  context_cache_config ────► ContextCacheConfig            │
│    (caches LLM prompt content across invocations)         │
│                                                           │
│  events_compaction_config ► EventsCompactionConfig        │
│    (summarizes older events to shrink context)            │
└──────────────────────────────────────────────────────────┘
```

- **Context Caching** tells the Gemini API to cache the static portions of the prompt (system instructions, examples, conversation prefix) so they are not re-tokenized on every invocation. Controlled by `ContextCacheConfig(min_tokens, ttl_seconds, cache_intervals)`.
- **Context Compression** uses a sliding window approach: after every N invocations (`compaction_interval`), older events are summarized by an LLM into a single compacted event. The `overlap_size` parameter controls how many invocations from the previous window are included in the new summary for continuity.

### Artifacts as Versioned Binary Storage

Artifacts store binary data (images, PDFs, audio, generated files) associated with a session. Each artifact is identified by a filename string and automatically versioned (integer versions starting at 0). Artifacts use `google.genai.types.Part` objects containing binary `inline_data` with a `mime_type`. The `ArtifactService` is registered on the `Runner` and accessed via `CallbackContext` or `ToolContext` methods.

---

## When to Use

- When a custom agent (`BaseAgent._run_async_impl`) needs direct access to the session, services, or control over the invocation (use `InvocationContext`).
- When an instruction provider needs to read state to dynamically generate agent instructions (use `ReadonlyContext`).
- When a callback needs to modify state or save/load artifacts before or after agent/model execution (use `CallbackContext`).
- When a tool function needs to authenticate against external APIs, search memory, list artifacts, or access its `function_call_id` (use `ToolContext`).
- When agents have large, static system instructions that are repeated across invocations and you want to reduce latency and token costs (use `ContextCacheConfig` with `static_instruction`).
- When agents have long-running conversations that risk exceeding the model's context window, and older events can be summarized without losing critical information (use `EventsCompactionConfig`).
- When you need to inspect, filter, or process the stream of events from `runner.run_async()` — detecting final responses, tool calls, state changes, or control signals (use `Event` and `EventActions`).
- When an agent needs to produce, store, and later retrieve binary files (images, reports, data exports) within a session (use `ArtifactService` with `save_artifact` / `load_artifact`).

---

## Rules

### Context Objects
- `InvocationContext` is created by the Runner and passed to `BaseAgent._run_async_impl(ctx)`. You never construct it manually.
- `ReadonlyContext` provides a read-only view of `state`. Writing to it has no effect or raises an error.
- In the current API, `CallbackContext` and `ToolContext` are unified into `google.adk.agents.Context`. The class exposes all capabilities, but tool-specific methods (`request_credential`, `request_confirmation`) require `function_call_id` to be set (automatic for tools, `None` for callbacks).
- For tool functions, `tool_context: ToolContext` (or `ctx: Context`) must be the **last** parameter in the function signature. It is auto-injected by the framework and excluded from the LLM schema.
- `CallbackContext` allows mutable `state` access. Changes made via `callback_context.state["key"] = value` are tracked and included in the `state_delta` of the next event.
- `ToolContext` extends `CallbackContext` capabilities with `request_credential()`, `get_auth_response()`, `search_memory()`, `list_artifacts()`, and `function_call_id`.
- All context objects expose `invocation_id` and `agent_name` for logging and tracking.

### Context Caching
- `ContextCacheConfig` is set on the `App` object via `context_cache_config=ContextCacheConfig(...)`.
- `min_tokens` (default 0): minimum estimated request tokens required to enable caching. Requests below this threshold are not cached.
- `ttl_seconds` (default 1800): time-to-live for the cache entry in seconds.
- `cache_intervals` (default 10, range 1–100): maximum number of invocations to reuse the same cache before forcing a refresh.
- Caching is most effective when combined with `static_instruction` on `LlmAgent`, which separates static content (cached) from dynamic `instruction` content (not cached).
- Requires Gemini 2.0 or higher models.

### Context Compression
- `EventsCompactionConfig` is set on the `App` object via `events_compaction_config=EventsCompactionConfig(...)`.
- `compaction_interval` (required): number of new invocations between compaction runs (e.g., 3 means compact after every 3 invocations).
- `overlap_size` (required): number of invocations from the previous window included in the next summary for continuity.
- The default summarizer uses the root agent's LLM model. If the root agent is a workflow agent (e.g., `SequentialAgent`) without a model, you must provide a `summarizer` parameter with an `LlmEventSummarizer(llm=...)` instance.
- `LlmEventSummarizer` also accepts a `prompt_template` parameter for customizing the summarization prompt.
- The Runner handles compaction automatically in the background at the end of each session interval.

### Events
- `Event` objects are immutable records with `author`, `id`, `invocation_id`, `timestamp`, `content`, and `actions`.
- `EventActions` carries side-effect signals: `state_delta` (dict of state changes), `artifact_delta` (dict of filename → version), `transfer_to_agent` (str), `escalate` (bool), `skip_summarization` (bool), `requested_auth_configs`, `requested_tool_confirmations`, `compaction`, `end_of_agent`.
- `state_delta` changes are only committed after the Runner processes the yielded event — not at the time state is modified in context.
- `event.is_final_response()` returns `True` for events that should be displayed to the user as the agent's answer.
- `event.get_function_calls()` returns a list of `FunctionCall` objects if the event contains tool invocation requests.
- `event.get_function_responses()` returns a list of `FunctionResponse` objects if the event contains tool results.
- `event.partial` indicates incomplete content chunks during streaming.

### Artifacts
- `BaseArtifactService` is the abstract interface. Implementations: `InMemoryArtifactService` (in-memory, non-persistent), `GcsArtifactService` (Google Cloud Storage, persistent).
- The `ArtifactService` must be passed to `Runner(artifact_service=...)` during initialization.
- `save_artifact(filename, artifact)` stores a `types.Part` object and returns the integer version number. The `artifact_delta` is recorded in the event's `EventActions`.
- `load_artifact(filename)` returns the latest version as `Optional[types.Part]`. Returns `None` if no artifact exists with that name.
- `list_artifacts()` returns a list of filenames available in the session (available on `ToolContext` only).
- Artifact filenames with a `user:` prefix are scoped to the user across sessions. Filenames without prefix are session-scoped.
- GCS path structure: `{app}/{user}/{session}/{filename}/{version}` for session-scoped; `{app}/{user}/user-scoped/{filename}/{version}` for user-scoped.

---

## Example

### Complete Example: Context Objects, Events, and Artifacts with Runner

```python
import asyncio
from google.adk.agents import LlmAgent
from google.adk.sessions import InMemorySessionService
from google.adk.artifacts import InMemoryArtifactService
from google.adk.runners import Runner
from google.adk.tools import ToolContext
from google.genai import types

APP_NAME = "context_demo"
USER_ID = "user1"
MODEL = "gemini-2.0-flash"


def generate_report(topic: str, tool_context: ToolContext) -> dict:
    """Generates a text report on the given topic and saves it as an artifact.

    Args:
        topic: The topic for the report.
        tool_context: Provided by ADK framework.

    Returns:
        Status of the report generation.
    """
    report_text = f"Report on {topic}: This is a comprehensive analysis."
    artifact_part = types.Part.from_text(text=report_text)

    # Save the report as an artifact (auto-versioned)
    version = tool_context.save_artifact(
        filename="report.txt", artifact=artifact_part
    )

    # Update state to track the last report
    tool_context.state["temp:last_report_topic"] = topic
    tool_context.state["temp:last_report_version"] = version

    # List all available artifacts
    artifacts = tool_context.list_artifacts()

    return {
        "status": "success",
        "filename": "report.txt",
        "version": version,
        "available_artifacts": artifacts,
    }


report_agent = LlmAgent(
    model=MODEL,
    name="ReportAgent",
    instruction=(
        "You are a report generation assistant. Use the generate_report "
        "tool when asked to create reports on any topic."
    ),
    tools=[generate_report],
)

session_service = InMemorySessionService()
artifact_service = InMemoryArtifactService()

runner = Runner(
    agent=report_agent,
    app_name=APP_NAME,
    session_service=session_service,
    artifact_service=artifact_service,  # Required for artifact support
)


async def main():
    session = await session_service.create_session(
        app_name=APP_NAME, user_id=USER_ID, session_id="s1"
    )

    msg = types.Content(
        parts=[types.Part(text="Generate a report on AI safety")], role="user"
    )

    async for event in runner.run_async(
        user_id=USER_ID, session_id="s1", new_message=msg
    ):
        # Inspect event metadata
        print(f"[Event] Author: {event.author}, ID: {event.id}")

        # Check for function calls (tool requests from LLM)
        function_calls = event.get_function_calls()
        if function_calls:
            for fc in function_calls:
                print(f"  Tool call: {fc.name}({fc.args})")

        # Check for state changes
        if event.actions and event.actions.state_delta:
            print(f"  State delta: {event.actions.state_delta}")

        # Check for artifact saves
        if event.actions and event.actions.artifact_delta:
            print(f"  Artifact delta: {event.actions.artifact_delta}")

        # Check for final response to user
        if event.is_final_response() and event.content and event.content.parts:
            print(f"  Final: {event.content.parts[0].text}")

    # Verify artifact was saved by loading it back
    loaded = await artifact_service.load_artifact(
        app_name=APP_NAME,
        user_id=USER_ID,
        session_id="s1",
        filename="report.txt",
    )
    if loaded:
        print(f"Loaded artifact: {loaded.text}")


# asyncio.run(main())
```

---

## Decision Table

| Need | Recommended Component | Why |
|---|---|---|
| Read state in a dynamic instruction provider | `ReadonlyContext` via `InstructionProvider` function | Read-only access prevents accidental mutations |
| Modify state in a callback | `CallbackContext.state["key"] = value` | Changes are tracked and included in `state_delta` |
| Save/load artifacts in a callback | `CallbackContext.save_artifact()` / `.load_artifact()` | Direct access to `ArtifactService` without manual wiring |
| Authenticate against an external API in a tool | `ToolContext.request_credential(auth_config)` | Triggers the interactive credential flow with `function_call_id` |
| Search long-term memory in a tool | `ToolContext.search_memory(query)` | Queries the configured `MemoryService` |
| List session artifacts in a tool | `ToolContext.list_artifacts()` | Returns all artifact filenames for the current session |
| Full session/service access in a custom agent | `InvocationContext` in `_run_async_impl` | Most comprehensive context; direct session and service access |
| Reduce cost for repeated large instructions | `ContextCacheConfig` on `App` + `static_instruction` on agent | Caches static prompt content across invocations |
| Shrink context for long conversations | `EventsCompactionConfig` on `App` | Summarizes older events to stay within token limits |
| Detect the user-facing response in the event stream | `event.is_final_response()` | Filters for events intended as the agent's answer |
| Detect tool invocations in the event stream | `event.get_function_calls()` | Returns `FunctionCall` objects from LLM output |
| Update state outside the Runner | `EventActions(state_delta={...})` + `session_service.append_event()` | Manual event injection for external state updates |
| Persistent binary file storage | `GcsArtifactService(bucket_name=...)` | Survives app restarts; stored in GCS |
| Temporary binary file storage for dev/test | `InMemoryArtifactService()` | Fast, no setup; lost on process exit |

---

## Common Patterns

### Pattern 1: Context Caching with Static Instructions

Separate large static content into `static_instruction` for caching, keeping `instruction` dynamic.

```python
from google.adk.agents import Agent
from google.adk.apps.app import App
from google.adk.agents.context_cache_config import ContextCacheConfig

root_agent = Agent(
    model="gemini-2.0-flash",
    name="PolicyAgent",
    static_instruction=(
        "You are a policy expert. Here is the full policy document: "
        "[... 10,000 words of policy text ...]"
    ),
    instruction="Answer the user's question about the policy concisely.",
)

app = App(
    name="policy-app",
    root_agent=root_agent,
    context_cache_config=ContextCacheConfig(
        min_tokens=2048,    # Only cache if context exceeds 2048 tokens
        ttl_seconds=600,    # Cache for 10 minutes
        cache_intervals=5,  # Refresh after 5 invocations
    ),
)
```

### Pattern 2: Context Compression for Long Conversations

Summarize older events every 3 invocations with 1 overlap.

```python
from google.adk.agents import Agent
from google.adk.apps.app import App, EventsCompactionConfig
from google.adk.apps.llm_event_summarizer import LlmEventSummarizer
from google.adk.models import Gemini

root_agent = Agent(
    model="gemini-2.0-flash",
    name="StoryAgent",
    instruction="You tell stories chapter by chapter. Continue when the user says 'continue'.",
)

summarizer = LlmEventSummarizer(llm=Gemini(model="gemini-2.5-flash"))

app = App(
    name="story-app",
    root_agent=root_agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,  # Compact every 3 invocations
        overlap_size=1,         # Include 1 invocation overlap
        summarizer=summarizer,  # Required if root is a workflow agent
    ),
)
```

### Pattern 3: Manual Event Injection for External State Updates

Update session state outside the Runner by creating and appending an event.

```python
import time
from google.adk.events import Event, EventActions

# Update state externally (e.g., from a webhook or background process)
state_changes = {
    "user:subscription_tier": "premium",
    "user:last_payment_ts": time.time(),
}
system_event = Event(
    invocation_id="external-update-001",
    author="system",
    actions=EventActions(state_delta=state_changes),
    timestamp=time.time(),
)
await session_service.append_event(session, system_event)
```

### Pattern 4: Processing Events from runner.run_async()

Filter and handle different event types from the Runner's event stream.

```python
from google.adk.events import Event

async for event in runner.run_async(
    user_id=USER_ID, session_id="s1", new_message=msg
):
    # Tool calls from LLM
    if event.get_function_calls():
        for fc in event.get_function_calls():
            print(f"Tool: {fc.name}")

    # Tool results
    if event.get_function_responses():
        for fr in event.get_function_responses():
            print(f"Tool result: {fr.name}")

    # Control flow signals
    if event.actions:
        if event.actions.transfer_to_agent:
            print(f"Transferring to: {event.actions.transfer_to_agent}")
        if event.actions.escalate:
            print("Escalation signaled")

    # Streaming partial content
    if event.partial and event.content:
        print(event.content.parts[0].text, end="", flush=True)

    # Final response
    if event.is_final_response() and event.content:
        print(f"\nFinal: {event.content.parts[0].text}")
```

### Pattern 5: Artifact Save and Load in a Callback

Save an artifact in a `before_model_callback` and load it later in a tool.

```python
from google.adk.agents.callback_context import CallbackContext
from google.adk.models import LlmRequest
from google.adk.tools import ToolContext
from google.genai import types
from typing import Optional
import json


def save_config_callback(
    callback_context: CallbackContext, request: LlmRequest
) -> Optional[types.Content]:
    """Saves a config artifact before each model call."""
    config = {"model_call_count": callback_context.state.get("model_calls", 0)}
    config_part = types.Part.from_text(text=json.dumps(config))
    callback_context.save_artifact(filename="config.json", artifact=config_part)
    callback_context.state["model_calls"] = config.get("model_call_count", 0) + 1
    return None  # Allow model call to proceed


def read_config_tool(tool_context: ToolContext) -> dict:
    """Reads the config artifact saved by the callback.

    Args:
        tool_context: Provided by ADK framework.

    Returns:
        The current configuration.
    """
    config_part = tool_context.load_artifact("config.json")
    if config_part and config_part.text:
        return {"config": json.loads(config_part.text)}
    return {"error": "No config artifact found"}
```

---

## Edge Cases

### State Changes Not Persisted Until Event Is Committed

When code modifies `context.state["key"] = value`, the change is recorded locally but not persisted until the event carrying the `state_delta` is yielded and processed by the Runner. If the process crashes between the state modification and the yield, the change is lost. Always verify critical state changes by reading from the session after the event is committed.

### ReadonlyContext State Mutations Silently Ignored

`ReadonlyContext` (used in `InstructionProvider` functions) provides a read-only view of state. Attempting to write `context.state["key"] = value` either raises an error or is silently ignored depending on the implementation. Never attempt state mutations in instruction providers.

### SequentialAgent Root with EventsCompactionConfig Missing Summarizer

If the `App`'s root agent is a `SequentialAgent`, `ParallelAgent`, or other workflow agent that has no `model` attribute, the default compaction logic fails because it cannot find a model for summarization. Provide an explicit `summarizer=LlmEventSummarizer(llm=Gemini(model="..."))` in the `EventsCompactionConfig`.

### InMemoryArtifactService Data Loss

`InMemoryArtifactService` stores all artifacts in process memory. All data is lost when the process exits. Use `GcsArtifactService` for any deployment that requires persistence across restarts.

### load_artifact Returns None for Missing Artifacts

`load_artifact(filename)` returns `None` (not an exception) if no artifact exists with the given filename. Always check for `None` before accessing the returned `Part`'s data.

### Artifact Delta Does Not Contain the Data

`event.actions.artifact_delta` is a dict mapping `filename → version` (int). It records *which* artifacts were saved and their version numbers — it does not contain the actual binary data. The data is stored separately by the `ArtifactService`.

### Context Caching with Small Requests

If a request's estimated token count is below `ContextCacheConfig.min_tokens`, caching is skipped entirely. Setting `min_tokens` too high means most requests bypass caching; setting it to 0 means all requests attempt caching (which has overhead for very small requests).

### is_final_response() on State-Only Events

Events that only carry `state_delta` (no `content`) from a `before_agent_callback` may unexpectedly return `True` for `is_final_response()` if they have no subsequent events. Always check that `event.content` is not `None` and `event.content.parts` is not empty before displaying to the user.

### Compaction Summarization Quality

Context compaction uses an LLM to summarize older events. If the summary loses important details (e.g., specific numbers, user preferences mentioned early in the conversation), the agent may produce incorrect responses in later turns. Tune `compaction_interval` and `overlap_size` to balance token savings with information retention. Use `LlmEventSummarizer(prompt_template=...)` to customize the summarization prompt for domain-specific needs.

### Calling Tool-Specific Methods in a Callback Context

Because `CallbackContext` and `ToolContext` are unified into `google.adk.agents.Context`, all methods are technically visible on the object. However, calling `request_credential()` or `request_confirmation()` inside a callback (where `function_call_id` is `None`) raises `ValueError`. Use `save_credential()` / `load_credential()` instead when working in callback scope.

---

## Failure Handling

### ArtifactService Not Configured

If no `artifact_service` is passed to the `Runner`, calling `save_artifact()` or `load_artifact()` from a context object raises a `ValueError` or returns `None`. Always register an `ArtifactService` when artifacts are needed.

### GcsArtifactService Permission Errors

If the GCS bucket does not exist or the service account lacks write permissions, `save_artifact()` raises an exception from the `google-cloud-storage` library. ADK catches this and returns the error to the LLM context. Validate bucket access at startup.

### Memory Service Not Configured

If `ToolContext.search_memory()` is called without a `MemoryService` registered on the Runner, a `ValueError` is raised. Check that `memory_service` is provided when agents or tools use memory search.

### Context Caching Failure

If the Gemini API rejects the caching request (unsupported model, quota exceeded, invalid config), the invocation proceeds without caching — it does not fail. The response will be slower and more expensive but still correct.

### Compaction Summarizer Error

If the summarization LLM call fails (network error, quota exceeded), the compaction is skipped for that interval. Events accumulate until the next successful compaction. Monitor logs for repeated compaction failures.

---

## Scalability Notes

### Context Caching Cost Reduction

Context caching can significantly reduce costs for agents with large system instructions (e.g., 10,000+ token policy documents). The Gemini API charges for cached tokens at a reduced rate. Savings increase with more invocations within the `ttl_seconds` window. The `cache_intervals` cap forces periodic refresh to pick up any configuration changes.

### Context Compression Token Savings

Without compaction, a 50-turn conversation sends all 50 turns of context to the LLM on every invocation. With `compaction_interval=3` and `overlap_size=1`, the context is kept to roughly 2–3 summary events plus the most recent invocations. Token savings grow linearly with conversation length.

### Artifact Storage Scaling

`InMemoryArtifactService` is process-scoped — no persistence, no sharing across instances. `GcsArtifactService` scales with GCS's built-in durability and throughput. For very large artifacts (videos, large datasets), consider streaming to external storage and storing only a reference URL in the artifact.

### Event Stream Volume

In complex multi-agent workflows, a single `runner.run_async()` call can yield dozens of events (tool calls, sub-agent transfers, streaming chunks). Processing overhead per event is minimal, but logging all events in production can create significant log volume. Filter events using `is_final_response()` and `get_function_calls()` to log only relevant events.

### Concurrent Invocations

The `InvocationContext` is per-invocation. Multiple concurrent invocations on the same session can cause race conditions in state. `InMemorySessionService` is not thread-safe for concurrent writes. Use `DatabaseSessionService` or `VertexAiSessionService` for concurrent production workloads.

---

## Anti-Patterns

| Anti-Pattern | Problem | Correct Approach |
|---|---|---|
| Constructing `InvocationContext` manually | Framework-internal object; manual construction bypasses Runner setup | Let the Runner create it via `runner.run_async()` |
| Writing state in a `ReadonlyContext` (instruction provider) | Writes are silently ignored or raise errors | Use `CallbackContext` or `ToolContext` for state mutations |
| Assuming state is persisted immediately after `context.state["key"] = value` | State is only committed after the event is yielded and processed by the Runner | Always yield an event after critical state changes before reading back |
| Using `InMemoryArtifactService` in production | All artifacts lost on process restart | Use `GcsArtifactService` for persistent storage |
| Not providing a `summarizer` to `EventsCompactionConfig` with workflow root agents | Compaction fails because the root agent has no model | Always pass `summarizer=LlmEventSummarizer(llm=...)` for workflow agents |
| Ignoring `None` from `load_artifact()` | `NoneType` access causes `AttributeError` | Always check for `None` before using the returned `Part` |
| Setting `compaction_interval=1` | Compacts on every invocation; excessive LLM calls and potential information loss | Use intervals of 3–5 for a reasonable balance |
| Caching with models below Gemini 2.0 | Context caching is not supported; may fail silently | Verify model compatibility before enabling caching |
| Processing all events in production without filtering | High log volume; unnecessary processing overhead | Filter with `is_final_response()` and `get_function_calls()` |
| Calling `request_credential()` in a callback | `function_call_id` is `None` in callbacks; raises `ValueError` | Use `save_credential()` / `load_credential()` in callback scope |

---

## Guidelines

1. **Use the narrowest context object for the task.** Use `ReadonlyContext` in instruction providers, `CallbackContext` in callbacks, and `ToolContext` in tools. Only use `InvocationContext` in `BaseAgent._run_async_impl` when full session/service access is genuinely needed.

2. **Combine `ContextCacheConfig` with `static_instruction` for maximum cache effectiveness.** Move large, unchanging content (policy documents, reference data, detailed persona descriptions) into `LlmAgent.static_instruction` and keep `instruction` for dynamic, per-turn content. This maximizes the cached portion of the prompt.

3. **Provide an explicit `LlmEventSummarizer` when using `EventsCompactionConfig`.** Even if the root agent has a model, explicitly setting the summarizer gives you control over which model does summarization and lets you use a cheaper/faster model for this ancillary task.

4. **Always register an `ArtifactService` on the `Runner` when agents produce binary outputs.** Pass `InMemoryArtifactService()` for development and `GcsArtifactService(bucket_name="...")` for production. Without it, `save_artifact()` and `load_artifact()` calls fail at runtime.

5. **Use `event.is_final_response()` to filter the event stream for user-facing content.** The Runner yields many events (tool calls, intermediate steps, state-only events). Only events where `is_final_response()` returns `True` and `event.content` is not `None` should typically be displayed to the user.

6. **Use `EventActions(state_delta=...)` with `session_service.append_event()` for external state updates.** When state needs to change outside the Runner (webhooks, background jobs), construct an `Event` with `state_delta` and append it directly. This ensures the change is recorded in the event history and persisted consistently.

7. **Set `min_tokens` on `ContextCacheConfig` to a value above your smallest expected request.** This avoids the overhead of caching for trivial requests. A value of 2048–4096 is a good starting point for most applications.

8. **Check `event.actions.artifact_delta` to detect artifact saves in the event stream.** This dict maps filenames to version numbers and can be used to trigger downstream processing (e.g., sending the artifact to a user, triggering a notification, or updating a UI).

---

## References

- [Official ADK Docs — Context](https://google.github.io/adk-docs/context/)
- [Official ADK Docs — Context Caching](https://google.github.io/adk-docs/context/caching/)
- [Official ADK Docs — Context Compression](https://google.github.io/adk-docs/context/compaction/)
- [Official ADK Docs — Events](https://google.github.io/adk-docs/events/)
- [Official ADK Docs — Artifacts](https://google.github.io/adk-docs/artifacts/)
- [Official ADK Docs — Event Loop](https://google.github.io/adk-docs/runtime/event-loop/)
- [Official ADK Docs — Apps](https://google.github.io/adk-docs/apps/)
- [ADK Python API Reference](https://google.github.io/adk-docs/api-reference/python/)
- [ADK Python GitHub](https://github.com/google/adk-python)
