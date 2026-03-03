---
name: vector-memory
description: >
  Use this skill when a Google ADK agent needs to store memory as vector
  embeddings and retrieve relevant memories by semantic similarity. Covers
  generating embeddings from text content, storing them in a vector store
  (Vertex AI Vector Search, ChromaDB, or in-memory), querying by cosine
  similarity, and injecting retrieved memories into agent context for
  retrieval-augmented reasoning.
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

This skill implements vector-based memory for Google ADK agents — storing memory
entries as embedding vectors and retrieving the most semantically relevant entries
by cosine similarity. Vector memory enables "find memories similar to this query"
rather than exact key-value lookups, making it ideal for large knowledge bases,
episodic memory retrieval, and RAG pipelines. This skill covers embedding generation,
vector store integration (Vertex AI Vector Search, ChromaDB, in-memory), similarity
search, and injecting retrieved memories into agent context.

## When to Use

- When the agent must retrieve the most relevant facts or past episodes from a
  large memory corpus (> 1000 entries) using semantic similarity.
- When exact keyword matching is insufficient and semantic relevance is needed.
- When implementing a RAG pipeline in an ADK agent.
- When building episodic memory retrieval ("find bookings similar to this route").
- When connecting the agent to a vector database as a long-term knowledge store.

## When NOT to Use

- Do not use vector memory for small memory sets (< 100 entries) — exact key-value
  lookup in `tool_context.state` is simpler and faster.
- Do not apply vector search when exact field matching is needed (e.g., lookup
  by order_id) — use a structured database query.
- Do not embed and index every tool call result — only index content that will
  be retrieved again based on semantic similarity.

## Google ADK Context

- **`tool_context.state["user:vector_memory_refs"]`**: Stores vector memory entry
  IDs in state, linking the ADK session to the external vector store.
- **`tool_context.user_id`**: Used as a namespace for user-specific vector entries.
- **Embedding model**: Use `google.generativeai.embed_content()` or the
  `text-embedding-004` model for generating embeddings from text in ADK pipelines.
- **Vector store options**:
  - **Vertex AI Vector Search**: GCP-native, scalable, managed. Production-recommended.
  - **ChromaDB**: Open-source, embeddable, no external service for small deployments.
  - **In-memory (numpy)**: Development/testing only.
- **Retrieval in ADK pipeline**: A tool function performs the vector search and
  returns the top-k relevant memory entries. The agent's instruction then reads
  these from state or the tool return value.
- **ADK Agent Execution Lifecycle**: Vector retrieval happens inside a tool function
  called by the LLM agent. Results are returned to the agent and/or stored in state.

## Capabilities

- Generates text embeddings using Google embedding models.
- Stores embedded memory entries in a vector store.
- Retrieves top-k semantically similar memories by cosine similarity.
- Namespaces vector entries by user_id for user-specific memory.
- Injects retrieved memories into agent context via state injection.
- Integrates with Vertex AI Vector Search, ChromaDB, and in-memory stores.

## Step-by-Step Instructions

1. **Choose a vector store** based on scale:
   - Dev/small: ChromaDB (local) or in-memory (numpy)
   - Production: Vertex AI Vector Search

2. **Generate embeddings** for memory content:
   ```python
   import google.generativeai as genai
   def embed_text(text: str) -> list[float]:
       result = genai.embed_content(model="models/text-embedding-004", content=text)
       return result["embedding"]
   ```

3. **Store a memory entry** in the vector store:
   ```python
   # ChromaDB example
   collection.add(
       ids=[memory_id],
       documents=[text],
       embeddings=[embed_text(text)],
       metadatas=[{"user_id": user_id, "type": memory_type, "timestamp": time.time()}],
   )
   ```

4. **Retrieve top-k relevant memories** from a tool:
   ```python
   def retrieve_relevant_memory(query: str, k: int, tool_context: ToolContext) -> dict:
       query_embedding = embed_text(query)
       results = collection.query(query_embeddings=[query_embedding], n_results=k,
                                   where={"user_id": tool_context.user_id})
       return {"memories": results["documents"][0] if results["documents"] else []}
   ```

5. **Inject retrieved memories** into agent context via state:
   ```python
   tool_context.state["temp:retrieved_memories"] = "\n".join(results["memories"])
   ```

6. **Reference in instruction**: `"Relevant memories:\n{temp:retrieved_memories?}"`

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "semantic",
  "content": "User prefers direct flights and aisle seats",
  "metadata": {"type": "preference", "source": "booking_tool"},
  "embedding": [0.12, -0.34, 0.56]
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "vmem_user42_001",
  "memory_retrieved": [
    {"content": "User prefers direct flights", "score": 0.94},
    {"content": "User prefers aisle seats", "score": 0.87}
  ],
  "memory_injected": True
}
```

## Error Handling

- **Embedding generation failure** (API error): Catch `google.api_core.exceptions.GoogleAPIError`.
  Retry once with exponential backoff. On persistent failure, skip embedding and
  fall back to keyword search.
- **Vector store connection failure**: Implement a fallback to session state
  keyword lookup. Log the error with context for debugging.
- **Low similarity scores** (all scores < 0.70): No sufficiently relevant memory
  found. Do not inject — return empty results and proceed without memory context.
- **Vector store collection not found**: Create the collection if it doesn't exist.
  Never raise on missing collection — initialize on demand.
- **Memory entry too long to embed** (> 8192 tokens): Chunk the content into
  segments and embed each separately. Store chunk IDs linked to the parent entry.

## Examples

### Example 1 — ChromaDB vector memory tool

```python
import time
import uuid
import chromadb
import google.generativeai as genai
from google.adk.tools import ToolContext

# Initialize ChromaDB client (persistent on disk)
chroma_client = chromadb.PersistentClient(path="./chroma_db")
memory_collection = chroma_client.get_or_create_collection(
    name="agent_memory",
    metadata={"hnsw:space": "cosine"},
)

def embed_text(text: str) -> list[float]:
    """Generates embedding for text using Google text-embedding-004."""
    result = genai.embed_content(model="models/text-embedding-004", content=text,
                                  task_type="retrieval_document")
    return result["embedding"]

def store_vector_memory(content: str, memory_type: str, tool_context: ToolContext) -> dict:
    """Stores a memory entry as a vector embedding in ChromaDB.

    Args:
        content (str): Text content to embed and store.
        memory_type (str): Memory category (e.g., preference, episode, fact).
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'memory_id' (str), 'stored' (bool).
    """
    memory_id = f"vmem_{tool_context.user_id}_{uuid.uuid4().hex[:8]}"
    embedding = embed_text(content)
    memory_collection.add(
        ids=[memory_id], documents=[content], embeddings=[embedding],
        metadatas=[{"user_id": tool_context.user_id, "type": memory_type, "timestamp": time.time()}],
    )
    return {"memory_id": memory_id, "stored": True, "content_length": len(content)}


def retrieve_vector_memory(query: str, top_k: int = 5, tool_context: ToolContext = None) -> dict:
    """Retrieves the top-k most semantically relevant memories for a query.

    Args:
        query (str): The query to find relevant memories for.
        top_k (int): Maximum number of results to return.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'memories' (list of dicts with content and score).
    """
    if not query.strip():
        return {"memories": [], "total": 0}
    query_embedding = embed_text(query)
    results = memory_collection.query(
        query_embeddings=[query_embedding], n_results=min(top_k, 10),
        where={"user_id": tool_context.user_id},
    )
    memories = []
    if results["documents"] and results["documents"][0]:
        for doc, dist, meta in zip(results["documents"][0],
                                    results["distances"][0],
                                    results["metadatas"][0]):
            score = round(1.0 - dist, 4)  # Convert cosine distance to similarity
            if score >= 0.70:  # Only return high-confidence results
                memories.append({"content": doc, "score": score, "type": meta.get("type", "unknown")})
    # Inject into state for instruction access
    if memories and tool_context:
        tool_context.state["temp:retrieved_memories"] = "\n".join(
            f"[{m['type']}|score={m['score']}]: {m['content']}" for m in memories
        )
    return {"memories": memories, "total": len(memories)}
```

### Example 2 — Agent with vector memory retrieval step

```python
from google.adk.agents import LlmAgent, SequentialAgent
from google.adk.agents.readonly_context import ReadonlyContext

def memory_augmented_instruction(ctx: ReadonlyContext) -> str:
    """Builds instruction that includes retrieved vector memories."""
    memories = ctx.state.get("temp:retrieved_memories", "")
    memory_section = f"\nRELEVANT MEMORIES:\n{memories}\n" if memories else ""
    return f"""
You are FlightBot for AcmeAir.{memory_section}
Use the relevant memories above to personalize your response.
Current booking step: {{booking_step?}}.
"""

agent = LlmAgent(
    name="memory_augmented_agent",
    model="gemini-2.0-flash",
    instruction=memory_augmented_instruction,
    tools=[retrieve_vector_memory, store_vector_memory, search_flights, process_payment],
)
```

## Edge Cases

- **Very short content** (< 5 words): Short content produces low-quality embeddings.
  Reject entries shorter than 20 characters.
- **Duplicate memories** (same content stored twice): Compute content hash before
  storing. Skip insertion if a duplicate hash already exists in the collection.
- **Collection size limit** (hardware-constrained): Implement a max collection size
  per user_id and apply eviction when exceeded.
- **Embedding model output dimension changes** (API update): Store the model name
  and dimension in collection metadata. Rebuild the collection on dimension change.

## Integration Notes

- **`google-adk-retrieval-augmented-generation`**: RAG is built on vector memory.
  This skill provides the vector storage and retrieval foundation.
- **`google-adk-memory-indexing`**: Complementary to vector indexing — covers
  metadata and keyword indexing on top of vector embeddings.
- **`google-adk-episodic-memory`**: Past episodes can be embedded and stored in
  the vector store for semantic episode retrieval.
- **`google-adk-memory-eviction-strategies`**: When the vector collection exceeds
  size limits, eviction removes old low-relevance entries.
