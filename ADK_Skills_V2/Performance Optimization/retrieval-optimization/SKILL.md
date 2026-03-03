---
name: retrieval-optimization
description: >
  Use this skill when optimizing the quality, speed, and cost of vector-based
  retrieval in Google ADK RAG pipelines. Covers tuning similarity thresholds,
  chunk size and overlap, top-k parameters, re-ranking strategies, hybrid
  retrieval (vector + keyword), query expansion, and retrieval result caching
  to maximize recall@k while minimizing retrieval latency.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: performance-optimization
---

## Purpose

This skill optimizes retrieval quality and performance in Google ADK RAG pipelines.
Poor retrieval is the most common cause of grounding failures in RAG agents:
wrong chunks retrieved → agent generates incorrect or hallucinated answers.
Optimization targets: (1) **quality** — recall@k, precision@k via threshold and
re-ranking tuning; (2) **speed** — reducing retrieval latency via efficient
indexing and caching; (3) **cost** — reducing embedding API calls via caching
and batch processing. This skill covers the complete retrieval optimization
workflow: chunk tuning, index configuration, similarity threshold selection,
hybrid retrieval, query expansion, and result re-ranking.

## When to Use

- When the RAG agent returns irrelevant or hallucinated answers despite having
  relevant content in the knowledge base.
- When retrieval latency is > 300ms and is a contributing factor to total agent latency.
- When the same query is retrieving identical results across turns (caching opportunity).
- When recall@k is < 0.80 on evaluation queries — chunks are not being retrieved.
- When keyword-critical queries fail with pure vector search.

## When NOT to Use

- Do not apply retrieval optimization for knowledge bases with < 100 documents —
  the full corpus can be injected directly.
- Do not raise the similarity threshold above 0.90 without testing — this will
  produce many false negatives (relevant chunks below threshold).
- Do not apply query expansion blindly — expanded queries can introduce noise for
  factual queries.

## Google ADK Context

- **Retrieval in ADK**: A tool function performs the vector search and injects
  results into `tool_context.state["temp:rag_context"]`.
- **Embedding model**: `models/text-embedding-004` — 768-dimensional, tuned for
  retrieval. Use `task_type="retrieval_query"` for query embeds; `"retrieval_document"`
  for indexing.
- **Similarity threshold**: `score = 1 - cosine_distance`. Only include chunks
  with score ≥ threshold. Start at 0.70; tune up/down based on recall.
- **Top-k**: Retrieve more (10–20) and re-rank to return fewer (3–5). Improves
  precision without sacrificing recall.
- **Chunk size**: 400–600 words is the recommended range. Smaller chunks = higher
  precision but lower recall (context split across chunks). Larger = higher recall
  but lower precision (noisy context).
- **Hybrid retrieval**: Combine vector similarity (semantic match) with BM25 keyword
  search. Use RRF (Reciprocal Rank Fusion) to merge results.
- **Query expansion**: Augment the user query with synonyms or rephrasing before
  embedding. Improves recall for domain-specific terminology.
- **Retrieval caching**: Cache results keyed by query embedding (or query hash)
  with TTL in `tool_context.state`.

## Capabilities

- Tunes similarity threshold for precision/recall balance.
- Tunes chunk size and overlap for retrieval quality.
- Implements top-k over-retrieval + re-ranking for higher precision.
- Implements hybrid vector + keyword retrieval with RRF merging.
- Implements query expansion for improved recall.
- Caches retrieval results to eliminate repeated vector store calls.
- Measures recall@k and precision@k on an evaluation query set.

## Step-by-Step Instructions

1. **Evaluate baseline retrieval quality**:
   - Create 20 evaluation queries with known ground-truth chunks.
   - Measure `recall@5` = fraction of ground-truth chunks in top-5 results.

2. **Tune chunk size**: Re-index at chunk sizes [256, 400, 512, 768] words.
   Compare `recall@5` at each size. Select the best.

3. **Tune similarity threshold**: Test thresholds [0.65, 0.70, 0.75, 0.80, 0.85].
   Pick the threshold that maximizes recall without adding too much noise.

4. **Implement over-retrieval + re-ranking**:
   ```python
   # Retrieve top-20, re-rank by cross-encoder score, return top-5
   results = collection.query(query_embeddings=[q_emb], n_results=20)
   # Re-rank by keyword overlap + score
   re_ranked = sorted(results, key=lambda r: rerank_score(r, query), reverse=True)[:5]
   ```

5. **Add hybrid keyword retrieval** using BM25 alongside vector search:
   ```python
   vector_results = vector_search(query, k=10)
   keyword_results = bm25_search(query, k=10)
   merged = rrf_merge(vector_results, keyword_results, k=5)
   ```

6. **Cache retrieval results** to avoid repeated embedding + vector search:
   ```python
   cache_key = f"temp:rag:{hashlib.md5(query.encode()).hexdigest()[:10]}"
   if not tool_context.state.get(cache_key):
       tool_context.state[cache_key] = perform_retrieval(query)
   ```

7. **Apply query expansion** for low-recall domains:
   ```python
   expanded = f"{query} synonyms: {get_synonyms(query)}"
   ```

## Input Format

```python
{
  "operation": "rag_retrieval",
  "execution_time_ms": 480,
  "token_usage": 0,
  "memory_usage_mb": 32,
  "cost_estimate": 0.0001,
  "retrieval_latency_ms": 450,
  "cache_available": True,
  "parallelizable": False,
  "context_size_tokens": 3000
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "threshold_tuning + hybrid_retrieval + result_caching",
  "performance_improvement": {
    "latency_reduction_ms": 380,
    "token_reduction": 800,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.00008,
    "throughput_improvement": 3.2
  }
}
```

## Error Handling

- **recall@k < 0.70** despite threshold tuning: Lower the threshold to 0.60 and
  add query expansion. If still failing, re-chunk the documents with more overlap.
- **Retrieval too slow** (> 500ms): Use an ANN index (HNSW in ChromaDB or Vertex AI
  Matching Engine) instead of brute-force. Enable retrieval result caching.
- **Cache invalidation needed** (knowledge base updated): Increment `app:kb_version`
  on update. Include the version in the cache key.
- **Hybrid retrieval produces duplicate results** (same chunk from both vector and
  keyword): Dedup by chunk ID after RRF merge.
- **Query expansion introduces noise**: Test query expansion on evaluation set.
  Only apply expansion if recall@k improves.

## Examples

### Example 1 — Optimized retrieval tool with caching and re-ranking

```python
import hashlib
import time
import chromadb
import google.generativeai as genai
from google.adk.tools import ToolContext

chroma_client = chromadb.PersistentClient(path="./rag_kb")
collection = chroma_client.get_or_create_collection("policies", metadata={"hnsw:space": "cosine"})

SIMILARITY_THRESHOLD = 0.72
TOP_K_RETRIEVE = 15
TOP_K_RETURN = 5
CACHE_TTL = 600  # 10 minutes

def optimized_retrieve(query: str, tool_context: ToolContext) -> dict:
    """Retrieves relevant chunks with caching, threshold filtering, and re-ranking.

    Args:
        query (str): User query to retrieve relevant knowledge for.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'chunks_retrieved' (int), 'sources' (list).
    """
    # 1. Cache check
    cache_key = f"temp:rag:{hashlib.md5(query.lower().encode()).hexdigest()[:10]}"
    cached = tool_context.state.get(cache_key)
    if cached and isinstance(cached, dict):
        tool_context.state["temp:rag_context"] = cached.get("context", "")
        return {"chunks_retrieved": cached.get("count", 0), "cached": True}

    # 2. Generate query embedding
    q_emb = genai.embed_content(
        model="models/text-embedding-004", content=query, task_type="retrieval_query"
    )["embedding"]

    # 3. Over-retrieve for re-ranking
    results = collection.query(query_embeddings=[q_emb], n_results=TOP_K_RETRIEVE)
    docs = results["documents"][0] if results["documents"] else []
    metas = results["metadatas"][0] if results["metadatas"] else []
    dists = results["distances"][0] if results["distances"] else []

    # 4. Filter by threshold and re-rank by keyword overlap
    candidates = []
    for doc, meta, dist in zip(docs, metas, dists):
        score = round(1.0 - dist, 4)
        if score >= SIMILARITY_THRESHOLD:
            # Simple re-rank: keyword overlap bonus
            query_words = set(query.lower().split())
            overlap = len(query_words & set(doc.lower().split()))
            final_score = score + overlap * 0.01
            candidates.append((final_score, doc, meta))

    candidates.sort(reverse=True)
    top = candidates[:TOP_K_RETURN]

    # 5. Assemble context
    context_parts = [f"[{m.get('source', 'kb')}]\n{d[:600]}" for _, d, m in top]
    rag_context = "\n\n---\n\n".join(context_parts) or "(No relevant context found)"
    tool_context.state["temp:rag_context"] = rag_context

    # 6. Cache the result
    cache_value = {"context": rag_context, "count": len(top)}
    tool_context.state[cache_key] = cache_value
    return {"chunks_retrieved": len(top), "cached": False, "sources": [m.get("source") for _, _, m in top]}
```

## Edge Cases

- **Empty knowledge base**: Return "(Knowledge base is empty)" and log a warning.
- **Query too long for embedding model** (> 8192 tokens): Truncate the query to
  the first 2000 chars. The most important keywords are usually at the start.
- **Very short chunks** (< 20 words): Merge short adjacent chunks at index time
  to produce semantically coherent chunks.
- **Knowledge base update race condition** (indexing while retrieval is running):
  Use a version-controlled collection. Swap to the new collection atomically after indexing.

## Integration Notes

- **`google-adk-retrieval-augmented-generation`**: This skill optimizes the retrieval
  stage of the full RAG pipeline.
- **`google-adk-vector-memory`**: The vector store infrastructure backing retrieval.
- **`google-adk-caching-strategies`**: Retrieval result caching applies the general
  caching patterns to the specific case of vector search results.
- **`google-adk-context-optimization`**: Retrieved chunks are injected into context.
  Retrieval optimization ensures only high-quality, relevant chunks are included.
