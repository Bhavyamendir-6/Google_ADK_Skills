---
name: semantic-memory
description: >
  Use this skill when a Google ADK agent needs to store and retrieve factual
  knowledge, domain facts, entity relationships, and general world knowledge
  that is independent of any specific user session or episode. Covers storing
  semantic knowledge in tool_context.state app: keys or external knowledge bases,
  and retrieving relevant facts for augmenting agent reasoning.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: memory-management
---

## Purpose

This skill implements semantic memory for Google ADK agents — the storage and
retrieval of factual, general knowledge that is independent of any specific user
session or past event. Semantic memory holds domain facts (airline codes, airport
names, product catalog, FAQ answers), entity relationships (customer → account →
orders), and structured knowledge bases that inform agent reasoning across all
users and sessions. In ADK, semantic memory is stored in `app:` state (for
small, frequently accessed facts), in external structured databases, or in
vector stores for semantically similar fact retrieval.

## When to Use

- When the agent needs access to domain-specific facts that are not specific to
  any user (e.g., airport codes, product catalog, policy documents).
- When FAQ-style knowledge must be retrieved to answer user questions.
- When the agent must reason about entity relationships (e.g., "which airlines
  fly to LHR?").
- When building a domain knowledge base that all agents in a system can access.
- When the agent must know what tools can do — storing tool capability metadata
  as semantic memory.

## When NOT to Use

- Do not use semantic memory for user-specific data — use `user:` state or
  episodic memory.
- Do not store mutable, frequently-changing data in semantic memory — it is
  designed for stable, reference knowledge.
- Do not use semantic memory as a replacement for proper database integration
  when the knowledge base is large (> 10K facts).

## Google ADK Context

- **`app:` prefix**: `tool_context.state["app:key"]` is application-global state
  shared across all users and sessions. Ideal for small, frequently accessed
  semantic facts.
- **`tool_context.state["app:domain_facts"]`**: A dict or JSON-serialized
  knowledge base accessible to all agents.
- **External knowledge bases**: For large semantic memory, use Vertex AI Agent
  Builder, a SQL database, or a document retrieval system.
- **Vector stores**: For semantically indexed semantic memory (retrieve facts
  similar to a user query), integrate Vertex AI Vector Search or ChromaDB.
- **`InstructionProvider`**: Injects relevant semantic facts into agent instruction
  at runtime based on the current context.
- **ADK Agent Execution Lifecycle**: `app:` state is loaded at agent initialization
  and shared globally — perfect for infrequently-changing semantic knowledge.

## Capabilities

- Stores domain knowledge as structured facts in `app:` state.
- Retrieves factual knowledge by key, keyword, or semantic similarity.
- Injects relevant facts into agent instructions via `InstructionProvider`.
- Integrates with external knowledge bases and vector stores.
- Provides FAQ-style question-answer retrieval.
- Maintains entity-relationship mappings as semantic memory.

## Step-by-Step Instructions

1. **Populate semantic facts** in `app:` state at application startup:
   ```python
   app_state = {
       "app:airport_codes": {
           "JFK": "John F. Kennedy International, New York",
           "LHR": "Heathrow Airport, London",
       },
       "app:airline_routes": {"BA": ["JFK-LHR", "LHR-CDG"]},
       "app:refund_policy": "Refunds available within 24 hours of booking.",
   }
   session = await session_service.create_session(app_name="my_app", user_id=user_id, state=app_state)
   ```

2. **Retrieve semantic facts** from a tool:
   ```python
   def get_airport_info(airport_code: str, tool_context: ToolContext) -> dict:
       airports = tool_context.state.get("app:airport_codes", {})
       info = airports.get(airport_code.upper(), "Unknown airport")
       return {"airport_code": airport_code, "info": info}
   ```

3. **Inject relevant facts into the instruction** via `InstructionProvider`:
   ```python
   def instruction_with_facts(ctx: ReadonlyContext) -> str:
       policy = ctx.state.get("app:refund_policy", "Contact support for refund info.")
       return f"You are FlightBot.\nREFUND POLICY: {policy}\n"
   ```

4. **Retrieve semantically similar facts** using a vector store for large knowledge bases
   (see `google-adk-vector-memory`).

5. **Update semantic memory** when the knowledge base changes by updating `app:` state
   and propagating the change to the session service.

## Input Format

```python
{
  "session_id": "sess_any",
  "user_id": "user_any",
  "memory_type": "semantic",
  "content": "airport_code=JFK|John F. Kennedy International, New York",
  "metadata": {"category": "airport", "region": "us-east"},
  "embedding": [0.12, -0.34, 0.56]
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "app:airport_codes:JFK",
  "memory_retrieved": [{"code": "JFK", "name": "John F. Kennedy International, New York"}],
  "memory_injected": True
}
```

## Error Handling

- **Missing `app:` fact** (key not in state): Return a safe default. Log a warning
  to populate the fact in the knowledge base.
- **Stale semantic fact** (e.g., policy changed): Implement a `refresh_semantic_memory`
  tool that updates `app:` state from the source of truth.
- **`app:` state too large** (> 100KB): Move the knowledge base to an external
  store. Store only an index or summary in `app:` state.
- **Schema drift** (semantic fact format changed): Version your semantic memory
  schema: `app:airport_codes_v2`.

## Examples

### Example 1 — Domain knowledge base in app: state

```python
from google.adk.tools import ToolContext

SEMANTIC_FACTS = {
    "app:refund_policy": "Full refund within 24h of booking. 50% refund within 7 days. No refund after.",
    "app:baggage_policy": "1 carry-on (7kg) and 1 checked bag (23kg) included in all fares.",
    "app:airport_codes": {"JFK": "New York Kennedy", "LHR": "London Heathrow", "CDG": "Paris Charles de Gaulle"},
}

def get_policy_info(policy_type: str, tool_context: ToolContext) -> dict:
    """Retrieves a specific policy from semantic memory.

    Args:
        policy_type (str): One of: refund, baggage.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'policy' (str).
    """
    key = f"app:{policy_type}_policy"
    policy = tool_context.state.get(key, "Policy information not available. Please contact support.")
    return {"policy_type": policy_type, "policy": policy}
```

### Example 2 — Entity relationship lookup

```python
from google.adk.tools import ToolContext

def get_airlines_for_route(origin: str, destination: str, tool_context: ToolContext) -> dict:
    """Looks up which airlines serve a given route from semantic memory.

    Args:
        origin (str): Origin airport code.
        destination (str): Destination airport code.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'airlines' (list).
    """
    route_map = tool_context.state.get("app:airline_routes", {})
    route_key = f"{origin.upper()}-{destination.upper()}"
    airlines = route_map.get(route_key, [])
    return {"route": route_key, "airlines": airlines, "found": len(airlines) > 0}
```

## Edge Cases

- **Semantic fact conflicts** (two entries for the same airport code): Implement
  a dedup/override policy. Later entries override earlier ones.
- **Missing knowledge base at startup**: Fail fast at agent initialization with
  a clear error message. Semantic memory must be populated before agents serve users.
- **Knowledge base update required mid-deployment**: Update `app:` state via an
  admin endpoint. Consider a version bump to invalidate stale cached facts.

## Integration Notes

- **`google-adk-vector-memory`**: For large knowledge bases (> 10K facts),
  store embeddings in a vector store and retrieve by semantic similarity.
- **`google-adk-retrieval-augmented-generation`**: RAG is the production pattern
  for large-scale semantic memory retrieval: embed query → vector search → inject facts.
- **`google-adk-memory-indexing`**: Indexing structure for `app:` state knowledge
  bases enables efficient non-vector keyword retrieval.
- **`google-adk-context-window-management`**: Inject only the relevant subset of
  semantic facts per turn — avoid dumping the entire knowledge base into context.
