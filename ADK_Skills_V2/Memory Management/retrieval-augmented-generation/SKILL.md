---
name: retrieval-augmented-generation
description: >
  Use this skill when implementing Retrieval-Augmented Generation (RAG) in a
  Google ADK agent pipeline. Covers chunking source documents, generating and
  storing embeddings in a vector store, implementing a retrieval tool that
  queries by semantic similarity, and injecting retrieved context into an
  LlmAgent's instruction for grounded, knowledge-augmented generation.
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

This skill implements Retrieval-Augmented Generation (RAG) for Google ADK agents —
enabling agents to answer questions grounded in a knowledge base of documents,
policies, manuals, or historical records without loading the entire corpus into
the context window. RAG augments LLM generation by: (1) chunking and embedding
source documents into a vector store at index time; (2) retrieving the top-k most
semantically relevant chunks at query time; and (3) injecting those chunks into
the LlmAgent's context for grounded generation. This skill covers the complete
ADK-native RAG pipeline from document ingestion to response generation.

## When to Use

- When the agent must answer questions grounded in a large document corpus
  (policy manuals, product catalogs, technical docs, past conversations).
- When the LLM's context window is too small to include the entire knowledge base.
- When hallucination must be reduced by grounding responses in retrieved source evidence.
- When implementing a customer support agent that references a KB of FAQ documents.
- When building an enterprise knowledge assistant with access to internal documentation.

## When NOT to Use

- Do not use RAG when the entire knowledge base fits in the context window
  (< 100K tokens) — direct injection is simpler and more reliable.
- Do not use RAG for structured data queries (e.g., "what is order #12345?") —
  use a structured database query tool.
- Do not apply RAG if the retrieved context is always the same regardless of
  the query — pre-load it directly into the instruction.

## Google ADK Context

- **RAG stages in ADK**:
  1. **Index**: Chunk documents → embed → store in vector store (Vertex AI/ChromaDB).
  2. **Retrieve**: Tool function queries vector store by embedding the user query.
  3. **Augment**: Inject top-k retrieved chunks into `session.state["temp:rag_context"]`.
  4. **Generate**: `LlmAgent` generates a response grounded in `{temp:rag_context?}`.
- **Retrieval tool**: A `FunctionTool` called by the `LlmAgent` that performs
  the vector search and stores results in state.
- **`tool_context.state["temp:rag_context"]`**: The injected retrieved context.
  Uses `temp:` prefix for auto-clearing after each turn.
- **Vertex AI Agent Builder / Vertex AI RAG Engine**: GCP-native managed RAG
  service that handles chunking, embedding, indexing, and retrieval.
- **Grounding instructions**: Add explicit grounding rules to the instruction:
  "Base your answer ONLY on the retrieved context. Do not add information not
  in the context."
- **ADK `LlmAgent.tools`**: The retrieval function is a tool called by the agent.
  The agent calls it autonomously when it determines retrieval is needed.

## Capabilities

- Chunks source documents for embedding and indexing.
- Generates text embeddings using Google text-embedding-004.
- Stores document chunks in a vector store (ChromaDB or Vertex AI Vector Search).
- Implements a retrieval tool function callable by LlmAgent.
- Injects top-k retrieved chunks into agent context via state injection.
- Supports source attribution (document name, chunk ID) in retrieved results.
- Implements grounding instructions to reduce hallucination.

## Step-by-Step Instructions

1. **Chunk documents** (at index time):
   ```python
   def chunk_document(text: str, chunk_size: int = 512, overlap: int = 50) -> list[str]:
       words = text.split()
       chunks = []
       for i in range(0, len(words), chunk_size - overlap):
           chunks.append(" ".join(words[i:i + chunk_size]))
       return chunks
   ```

2. **Embed and store chunks** in the vector store:
   ```python
   import google.generativeai as genai
   for idx, chunk in enumerate(chunks):
       embedding = genai.embed_content(model="models/text-embedding-004", content=chunk)["embedding"]
       collection.add(ids=[f"doc_{doc_id}_chunk_{idx}"], documents=[chunk],
                      embeddings=[embedding], metadatas=[{"source": doc_name, "chunk_idx": idx}])
   ```

3. **Implement a retrieval tool** as an ADK `FunctionTool`:
   ```python
   def retrieve_context(query: str, tool_context: ToolContext) -> dict:
       embedding = genai.embed_content(model="models/text-embedding-004", content=query,
                                       task_type="retrieval_query")["embedding"]
       results = collection.query(query_embeddings=[embedding], n_results=5)
       chunks = results["documents"][0] if results["documents"] else []
       sources = results["metadatas"][0] if results["metadatas"] else []
       rag_context = "\n\n".join(
           f"[Source: {s.get('source', 'unknown')}]\n{c}"
           for c, s in zip(chunks, sources)
       )
       tool_context.state["temp:rag_context"] = rag_context or "(No relevant context found)"
       return {"chunks_retrieved": len(chunks), "sources": [s.get("source") for s in sources]}
   ```

4. **Add retrieval tool to the agent** and include the context injection in the instruction:
   ```python
   agent = LlmAgent(
       name="rag_agent",
       model="gemini-2.0-flash",
       instruction="""You are a knowledge assistant.
       Retrieved context:
       {temp:rag_context?}
       Base your answer ONLY on the retrieved context above.
       If the context does not contain the answer, say "I don't have information on that."
       """,
       tools=[retrieve_context],
   )
   ```

5. **Evaluate grounding quality** using `response_match_score` in ADK evals.

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "semantic",
  "content": "query=What is the refund policy for international flights?",
  "metadata": {"k": 5, "collection": "airline_policies", "min_score": 0.70},
  "embedding": [0.12, -0.34, 0.56]
}
```

## Output Format

```python
{
  "memory_stored": False,
  "memory_id": None,
  "memory_retrieved": [
    {"content": "Refunds are available within 24 hours...", "source": "refund_policy.pdf", "score": 0.93},
    {"content": "For international flights, refunds are processed...", "source": "intl_policy.pdf", "score": 0.87}
  ],
  "memory_injected": True
}
```

## Error Handling

- **Vector store unavailable**: Fall back to a static knowledge base stored in
  `app:` state or a simple keyword search. Return "(Retrieval system unavailable)".
- **No relevant chunks found** (all scores < threshold): Return "(No relevant
  context found for your query)". Do not inject empty context — teach the agent
  to acknowledge gaps.
- **Retrieved chunks are too long** (exceed injection budget): Truncate each
  chunk to 500 words. Only inject top-3 chunks if all 5 together exceed the budget.
- **Embedding generation failure**: Retry once. On persistent failure, skip RAG
  and respond from LLM parametric knowledge with a disclaimer.
- **Source attribution lost** (no metadata in vector store): Store source document
  name and chunk_idx in metadata for every chunk on ingestion.
- **Context injection overwrites earlier state**: Use `temp:rag_context` to
  auto-clear and avoid stale context from previous turns.

## Examples

### Example 1 — Complete RAG pipeline for a policy KB agent

```python
import chromadb
import google.generativeai as genai
from google.adk.agents import LlmAgent
from google.adk.tools import ToolContext

# Initialize ChromaDB
chroma_client = chromadb.PersistentClient(path="./rag_kb")
policy_collection = chroma_client.get_or_create_collection(
    name="airline_policies", metadata={"hnsw:space": "cosine"}
)

# --- INDEX TIME ---
def index_policy_document(doc_name: str, doc_text: str, chunk_size: int = 400) -> int:
    """Chunks and indexes a policy document for RAG retrieval."""
    words = doc_text.split()
    chunks = [" ".join(words[i:i+chunk_size]) for i in range(0, len(words), chunk_size - 50)]
    for idx, chunk in enumerate(chunks):
        if len(chunk.strip()) < 20:
            continue
        chunk_id = f"{doc_name}_chunk_{idx}"
        embedding = genai.embed_content(
            model="models/text-embedding-004", content=chunk,
            task_type="retrieval_document"
        )["embedding"]
        policy_collection.add(
            ids=[chunk_id], documents=[chunk], embeddings=[embedding],
            metadatas=[{"source": doc_name, "chunk_idx": idx}],
        )
    return len(chunks)

# --- RETRIEVAL TOOL ---
def retrieve_policy_context(query: str, tool_context: ToolContext) -> dict:
    """Retrieves the most relevant policy chunks for the user's query.

    Args:
        query (str): The user's policy-related question.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'chunks_retrieved' (int) and 'sources' (list).
    """
    if not query.strip():
        tool_context.state["temp:rag_context"] = "(No query provided)"
        return {"chunks_retrieved": 0, "sources": []}

    query_embedding = genai.embed_content(
        model="models/text-embedding-004", content=query,
        task_type="retrieval_query"
    )["embedding"]
    results = policy_collection.query(
        query_embeddings=[query_embedding], n_results=5
    )
    chunks = results["documents"][0] if results["documents"] else []
    metadatas = results["metadatas"][0] if results["metadatas"] else []
    distances = results["distances"][0] if results["distances"] else []

    context_parts = []
    sources = []
    for chunk, meta, dist in zip(chunks, metadatas, distances):
        score = round(1.0 - dist, 3)
        if score < 0.65:
            continue  # Filter low-confidence chunks
        sources.append(meta.get("source", "unknown"))
        context_parts.append(f"[Source: {meta.get('source')} | Score: {score}]\n{chunk[:800]}")

    rag_context = "\n\n---\n\n".join(context_parts) if context_parts else "(No relevant policy found)"
    tool_context.state["temp:rag_context"] = rag_context
    return {"chunks_retrieved": len(context_parts), "sources": sources}

# --- RAG AGENT ---
RAG_INSTRUCTION = """
You are AcmeAir's policy assistant.

RETRIEVED POLICY CONTEXT:
{temp:rag_context?}

RULES:
1. Base your answer ONLY on the Retrieved Policy Context above.
2. Quote relevant sections directly when possible.
3. If the context does not contain the answer, respond: "I couldn't find specific policy information on that. Please contact support."
4. Always cite the source document name in your response.
"""

policy_agent = LlmAgent(
    name="policy_rag_agent",
    model="gemini-2.0-flash",
    instruction=RAG_INSTRUCTION,
    tools=[retrieve_policy_context],
)
```

## Edge Cases

- **Chunk boundary cuts a sentence**: Use sentence-aware chunking
  (split on `". "` rather than raw word count) to avoid mid-sentence breaks.
- **Very short documents** (< 100 words): Don't chunk — store as a single entry.
- **Duplicate chunks** (same section appears in multiple documents): Deduplicate
  by content hash during indexing.
- **Query contains no keywords** (very vague question): Return top-3 results by
  recency or frequency. Ask the user to refine.
- **Large knowledge base** (> 100K chunks): Use Vertex AI Vector Search instead
  of ChromaDB for scalable ANN search.

## Integration Notes

- **`google-adk-vector-memory`**: Provides the vector store infrastructure.
  RAG is the primary application of vector memory.
- **`google-adk-context-window-management`**: Limits how many retrieved chunks
  can fit in each invocation's context window.
- **`google-adk-semantic-memory`**: Domain facts stored in app: state are an
  alternative to RAG for small, structured knowledge bases.
- **`google-adk-memory-compression`**: Truncates retrieved chunks to fit the
  context window budget before injection.
