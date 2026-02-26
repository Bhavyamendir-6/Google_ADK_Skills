# Context Optimization — Detailed Reference

## Overview

As agent sessions grow longer, the context window fills with event history, slowing responses and increasing costs. ADK provides two complementary solutions:

- **Context Caching** — Cache repeated data (instructions, large datasets) to avoid resending every turn
- **Context Compression** — Summarize older events to keep the context window lean

## Context Caching

### What It Does

Caches request data with generative AI models that support it (Gemini 2.0+). When the same instructions or data are sent across multiple turns, the model reuses the cached version instead of reprocessing.

### Configuration

```python
from google.adk import Agent
from google.adk.apps.app import App
from google.adk.agents.context_cache_config import ContextCacheConfig

root_agent = Agent(
    model="gemini-2.5-flash",
    name="analysis_agent",
    instruction="""You are a data analyst. Given a dataset, answer questions about it.
    Always provide statistical summaries with your answers.
    Format outputs as markdown tables when presenting numerical data.""",
)

app = App(
    name="data-analysis-app",
    root_agent=root_agent,
    context_cache_config=ContextCacheConfig(
        min_tokens=2048,    # Only cache when request > 2048 tokens
        ttl_seconds=600,    # Cache lasts 10 minutes
        cache_intervals=5,  # Refresh after 5 uses
    ),
)
```

### Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `min_tokens` | `0` | Minimum tokens to trigger caching. Set higher to skip caching for small requests. |
| `ttl_seconds` | `1800` (30 min) | How long cached content lives before refresh. |
| `cache_intervals` | `10` | Max times cached content is reused before expiring. |

### When to Use

- Agent has **long, static instructions** that don't change between turns
- Agent processes **large documents or datasets** repeatedly
- Multiple turns use **similar prompt structures**
- You want to **reduce latency and token costs**

### Alternative: `static_instruction`

If your instructions never change during a session, consider using `static_instruction` instead of caching. This amends the system instructions for the model directly:

```python
agent = Agent(
    model="gemini-2.5-flash",
    name="pet_agent",
    instruction="Handle dynamic per-turn instructions here.",
    static_instruction="""You are a virtual pet care assistant.
    You know about dogs, cats, birds, and fish.
    Always be enthusiastic and use pet-related emojis.""",
)
```

---

## Context Compression (Event Compaction)

### What It Does

Uses a sliding window approach to summarize older events in the session history. When the event count reaches a threshold, older events are compressed into a summary, keeping the context window lean.

### Configuration

```python
from google.adk.apps.app import App, EventsCompactionConfig

app = App(
    name="long-running-agent",
    root_agent=root_agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,  # Compress every 3 invocations
        overlap_size=1,         # Include 1 prior event in next window
    ),
)
```

### How It Works

With `compaction_interval=3` and `overlap_size=1`:

```
Events 1-3 → Compressed into Summary A
Events 3-6 → Compressed into Summary B (event 3 overlaps)
Events 6-9 → Compressed into Summary C (event 6 overlaps)
```

The overlap ensures continuity between summaries — no context is lost at boundaries.

### Configuration Parameters

| Parameter | Description |
|-----------|-------------|
| `compaction_interval` | Number of invocations before compaction triggers |
| `overlap_size` | How many prior events are included in the new summary window |
| `summarizer` | Optional custom LLM summarizer |

### Custom Summarizer

Control which model performs summarization:

```python
from google.adk.apps.app import App, EventsCompactionConfig
from google.adk.apps.llm_event_summarizer import LlmEventSummarizer
from google.adk.models import Gemini

# Use a fast model for summarization
summarization_llm = Gemini(model="gemini-2.5-flash")
my_summarizer = LlmEventSummarizer(llm=summarization_llm)

app = App(
    name="efficient-agent",
    root_agent=root_agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=5,
        overlap_size=2,
        summarizer=my_summarizer,
    ),
)
```

### When to Use

- Agent sessions are **long-running** (many turns)
- Context window is **filling up**, causing slower responses
- Older events contain **repetitive or low-value information**
- You want to **maintain conversation coherence** while reducing token count

---

## Combining Both Techniques

For maximum efficiency, use caching AND compression together:

```python
from google.adk.apps.app import App, EventsCompactionConfig
from google.adk.agents.context_cache_config import ContextCacheConfig

app = App(
    name="optimized-agent",
    root_agent=root_agent,
    context_cache_config=ContextCacheConfig(
        min_tokens=1024,
        ttl_seconds=900,
        cache_intervals=10,
    ),
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=5,
        overlap_size=1,
    ),
)
# Caching avoids resending static content
# Compression keeps dynamic history lean
```

## Decision Matrix

| Symptom | Solution |
|---------|----------|
| Same large instructions sent every turn | **Context Caching** with appropriate `min_tokens` |
| Session has 20+ turns and getting slow | **Context Compression** with `compaction_interval=5` |
| Both large instructions AND long sessions | **Both** — caching + compression |
| Instructions never change | **`static_instruction`** parameter |
| Small requests, short sessions | **Neither** — overhead not worth it |
