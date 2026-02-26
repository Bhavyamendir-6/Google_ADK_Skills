---
name: memory-management
description: >
  Manage conversational context, state, and long-term memory in Google ADK agents.
  Covers Session lifecycle, State scoping with prefixes, MemoryService for cross-session
  knowledge, context caching for performance, and context compression for long-running
  sessions. Use when designing, implementing, or debugging any data persistence or
  context management in an ADK agent application.
metadata:
  author: adk-skills
  version: "1.0"
---

# Memory Management in Google ADK

## When to use this skill

Activate this skill when the task involves:

- Managing conversation history or agent context
- Storing and retrieving data across agent turns or sessions
- Choosing between State, Memory, or Artifacts for data persistence
- Implementing cross-session knowledge recall
- Optimizing context window usage (caching or compression)
- Debugging state not persisting, context growing too large, or memory search issues

## Core Concepts — The Three Layers

| Layer | Scope | Persistence | What It Stores |
|-------|-------|-------------|----------------|
| **Session** | Single conversation thread | Depends on `SessionService` | Event history + state |
| **State** (`session.state`) | Current session (or user/app/temp) | Depends on service + prefix | Key-value pairs |
| **Memory** (`MemoryService`) | Across all sessions | Depends on implementation | Searchable knowledge archive |

**Mental model:**
- **Session/State** = short-term memory during one chat
- **Memory** = searchable long-term archive from many past chats

## Session

A `Session` object tracks one conversation thread. Key properties:

- **`id`** — Unique conversation identifier
- **`app_name`** — Application identifier
- **`user_id`** — User identifier
- **`events`** — Chronological list of all interactions (messages, tool calls, responses)
- **`state`** — Key-value scratchpad for the current conversation
- **`last_update_time`** — Timestamp of last activity

### SessionService Implementations

| Implementation | Persistence | Best For |
|---------------|-------------|----------|
| `InMemorySessionService` | ❌ Lost on restart | Local dev / testing |
| `DatabaseSessionService` | ✅ Persistent | Production (self-hosted DB) |
| `VertexAiSessionService` | ✅ Persistent | Production (Google Cloud) |

### Lifecycle

1. **Create** — `session_service.create_session(app_name, user_id, session_id)`
2. **Run** — Runner gets session, agent processes events and state
3. **Update** — Runner calls `append_event()` to save interactions + state changes
4. **Delete** — `session_service.delete_session()` when conversation ends

## State (`session.state`)

State is a key-value dictionary scoped to the current session. Used for:

- Tracking task progress (`booking_step: "confirm_payment"`)
- Storing user preferences (`user:theme: "dark"`)
- Passing data between agents via `output_key`

### State Prefixes — Scope Matters

| Prefix | Scope | Persistence | Example |
|--------|-------|-------------|---------|
| *(none)* | Current session only | With persistent service | `current_intent` |
| `user:` | All sessions for this user | With persistent service | `user:preferred_language` |
| `app:` | All users, all sessions | With persistent service | `app:global_discount` |
| `temp:` | Current invocation only | ❌ Never persisted | `temp:raw_api_response` |

### State Update Rules

- ✅ **Always** update state through `CallbackContext.state` or `ToolContext.state`
- ✅ **Use** `output_key` on `LlmAgent` to auto-save agent responses to state
- ❌ **Never** directly modify `session.state` on a `SessionService`-retrieved session object
- Direct modification bypasses event tracking, breaks persistence, and is not thread-safe

### State Best Practices

- Store only essential, dynamic data (minimalism)
- Use basic serializable types only (strings, numbers, booleans, simple lists/dicts)
- Use clear key names with appropriate prefixes
- Avoid deep nesting; keep structures shallow

## Memory (`MemoryService`)

Memory is a searchable, cross-session knowledge store. Workflow:

1. **Ingest** — `memory_service.add_session_to_memory(session)` after session completes
2. **Search** — Agent uses `load_memory` tool to query past context
3. **Results** — `MemoryService` returns relevant snippets from past sessions

### Memory Tools

| Tool | Behavior |
|------|----------|
| `LoadMemory` | Agent decides when to search memory (on-demand) |
| `PreloadMemoryTool` | Automatically loads relevant memory at start of each turn |

### MemoryService Implementations

| Implementation | Search | Persistence | Best For |
|---------------|--------|-------------|----------|
| `InMemoryMemoryService` | Keyword matching | ❌ | Prototyping |
| `VertexAiMemoryBankService` | Semantic search | ✅ | Production |

## Context Optimization

### Context Caching

Cache repeated request data (instructions, large datasets) to avoid resending every turn.

Configure at the `App` level with `ContextCacheConfig`:
- **`min_tokens`** — Minimum token count to trigger caching (default: 0)
- **`ttl_seconds`** — Cache time-to-live (default: 1800 = 30 min)
- **`cache_intervals`** — Max uses before refresh (default: 10)

### Context Compression

Summarize older events to keep context window lean in long-running sessions.

Configure at the `App` level with `EventsCompactionConfig`:
- **`compaction_interval`** — Number of invocations before compaction triggers
- **`overlap_size`** — How many prior events to include in new summaries
- **`summarizer`** — Optional custom LLM summarizer

## Decision Guide

| Need | Use |
|------|-----|
| Data for this conversation only | **State** (no prefix) |
| User preferences across sessions | **State** (`user:` prefix) |
| App-wide settings | **State** (`app:` prefix) |
| Temporary calculation within one turn | **State** (`temp:` prefix) |
| Recall from past conversations | **Memory** (`MemoryService`) |
| Reduce repeated token costs | **Context Caching** |
| Keep long sessions performant | **Context Compression** |

## References

- See [State reference](references/STATE_REFERENCE.md) for detailed code examples on state management, prefixes, and update patterns.
- See [Memory reference](references/MEMORY_REFERENCE.md) for MemoryService setup, search tools, and cross-session recall examples.
- See [Context Optimization reference](references/CONTEXT_OPTIMIZATION_REFERENCE.md) for caching and compression configuration with code.

## Edge Cases

- **State lost on restart:** You're using `InMemorySessionService` — switch to `DatabaseSessionService` or `VertexAiSessionService` for persistence.
- **State not updating:** You're modifying `session.state` directly instead of through `CallbackContext.state` or `ToolContext.state` — the change bypasses event tracking.
- **Memory search returns nothing:** Session was never ingested — call `add_session_to_memory()` after session completes.
- **Context window growing too large:** Enable `EventsCompactionConfig` to summarize older events.
- **`temp:` state missing next turn:** By design — `temp:` prefix is scoped to the current invocation only.
- **Parallel agent state conflicts:** Use unique `output_key` values per sub-agent to avoid race conditions.
