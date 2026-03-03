---
name: memory-compression
description: >
  Use this skill when reducing the size of stored memories in a Google ADK agent
  to stay within context window limits and session state size limits. Covers
  truncation, lossless compression (deduplication, structural compaction), and
  lossy compression (LLM-based summarization of old memory) strategies for
  compressing tool_context.state entries.
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

This skill implements memory compression for Google ADK agents — reducing the
size of stored memory entries while preserving the most important information.
As sessions grow, tool results, episode logs, and injected context accumulate
and may exceed state size limits or context window budgets. This skill applies
three compression approaches: (1) truncation (simple character/token limits),
(2) lossless structural compaction (deduplication, field normalization), and
(3) LLM-based summarization (lossy but preserves semantic meaning). The goal
is to maintain high memory utility while respecting size constraints.

## When to Use

- When a session state value (e.g., a tool result or episode log) exceeds 5KB.
- When the context window budget for memory injection is exceeded.
- When the episode log grows beyond the eviction threshold.
- When tool results from search APIs return large payloads that need compaction
  before storage.
- When long-form agent outputs must be compressed into compact summaries for state storage.

## When NOT to Use

- Do not compress security-sensitive data (authentication tokens, PII) via LLM
  summarization — use field-aware truncation only.
- Do not apply lossy compression to data that requires exact recall (e.g., booking
  references, financial amounts, legal terms).
- Do not compress memory pre-emptively without first checking whether size limits
  are actually being approached.

## Google ADK Context

- **`tool_context.state` size limits**: Practical limit per value ~64KB for most
  backends. The injection token limit (what fits in the LLM context) is lower.
- **Truncation**: The simplest compression — `value[:max_chars]`. Preserves the
  beginning of content. Appropriate for structured data where the beginning is
  most important.
- **Structural compaction**: Remove duplicate entries from lists; normalize field
  names; merge similar records; remove nulls.
- **LLM summarization**: Call a separate `LlmAgent` or direct Gemini API call to
  summarize long text into a compact representation. Lossy but semantic-preserving.
- **`before_agent_callback`**: Can apply compression to state values before the
  LLM is invoked to ensure the context window is not exceeded.
- **Chunking**: For content too large to summarize in one call, split into chunks,
  compress each, and concatenate the compressed chunks.

## Capabilities

- Applies character/token truncation to state values.
- Performs structural deduplication of episode logs and result lists.
- Compresses long tool results before state storage.
- Applies LLM-based semantic compression to long text memory.
- Implements chunked compression for very large memory entries.
- Validates compressed output size before storing.

## Step-by-Step Instructions

1. **Detect oversized memory** before storing:
   ```python
   MAX_VALUE_CHARS = 4000
   if len(str(content)) > MAX_VALUE_CHARS:
       content = compress_memory(content)
   ```

2. **Apply truncation** for structured data (tool results, JSON):
   ```python
   def truncate_memory(content: str, max_chars: int = 2000) -> str:
       if len(content) <= max_chars:
           return content
       return content[:max_chars] + "... [truncated]"
   ```

3. **Apply structural compaction** for lists (episode logs, search results):
   ```python
   def compact_result_list(results: list, max_items: int = 20, key_fields: list = None) -> list:
       seen = set()
       compacted = []
       for item in results:
           fingerprint = str({k: item.get(k) for k in (key_fields or list(item.keys())[:3])})
           if fingerprint not in seen:
               seen.add(fingerprint)
               compacted.append(item)
       return compacted[:max_items]
   ```

4. **Apply LLM summarization** for long free-text memory:
   ```python
   import google.generativeai as genai
   def summarize_memory(text: str, max_words: int = 100) -> str:
       prompt = f"Summarize this in {max_words} words or fewer, preserving key facts:\n\n{text}"
       response = genai.generate_content(model="gemini-2.0-flash", contents=[{"text": prompt}])
       return response.text.strip()
   ```

5. **Validate compressed size**:
   ```python
   compressed = compress_memory(content)
   assert len(compressed) < MAX_VALUE_CHARS, "Compression insufficient"
   tool_context.state[key] = compressed
   ```

## Input Format

```python
{
  "session_id": "sess_abc123",
  "user_id": "user_42",
  "memory_type": "short_term",
  "content": "[...very long search result JSON with 500 items...]",
  "metadata": {"original_size_chars": 45000, "target_size_chars": 2000},
  "embedding": []
}
```

## Output Format

```python
{
  "memory_stored": True,
  "memory_id": "sess_abc123:search_results_compressed",
  "memory_retrieved": [{"compressed_size_chars": 1850, "original_size_chars": 45000}],
  "memory_injected": True
}
```

## Error Handling

- **LLM summarization removes critical facts**: Only use LLM compression for
  narrative text. For structured data (JSON, tabular), use truncation + compaction.
- **Compression ratio insufficient** (compressed still > limit): Apply multi-pass
  compression — first truncate, then summarize the truncated version.
- **Compressed content loses important keys**: For JSON compression, always
  preserve required fields explicitly before applying general truncation.
- **Compression tool call adds latency**: Cache compressed results in state.
  Don't re-compress the same content on every turn.
- **Empty content after compression**: Validate that compressed output is non-empty
  before storing. Return a placeholder: `"[Content compressed — no summary available]"`.

## Examples

### Example 1 — Compress tool result before state storage

```python
import json
from google.adk.tools import ToolContext

MAX_RESULT_CHARS = 3000

def search_and_compress(query: str, tool_context: ToolContext) -> dict:
    """Searches and stores results in state with compression for large responses.

    Args:
        query (str): Search query.
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'items_found' (int), 'compressed' (bool).
    """
    raw_results = _do_search(query)  # Returns list of dicts
    # Structural compaction: deduplicate and limit fields
    compacted = []
    seen_ids = set()
    for item in raw_results[:50]:  # Cap at 50 items
        item_id = item.get("id") or item.get("flight_number")
        if item_id and item_id in seen_ids:
            continue
        seen_ids.add(item_id)
        # Keep only essential fields
        compacted.append({k: item[k] for k in ["id", "airline", "price", "departs", "arrives"] if k in item})

    serialized = json.dumps(compacted, ensure_ascii=False)
    compressed = False
    if len(serialized) > MAX_RESULT_CHARS:
        # Further truncate by reducing items
        while len(serialized) > MAX_RESULT_CHARS and compacted:
            compacted.pop()  # Remove last item
            serialized = json.dumps(compacted, ensure_ascii=False)
        compressed = True

    tool_context.state["search_results"] = serialized
    return {"items_found": len(compacted), "compressed": compressed, "stored_chars": len(serialized)}

def _do_search(query):
    return []  # Placeholder
```

### Example 2 — LLM-based semantic memory compression

```python
import google.generativeai as genai
from google.adk.tools import ToolContext

def compress_session_narrative(tool_context: ToolContext) -> dict:
    """Compresses the session conversation narrative into a compact summary.

    Args:
        tool_context (ToolContext): ADK tool context.
    Returns:
        dict: Contains 'original_chars' (int), 'compressed_chars' (int).
    """
    narrative = tool_context.state.get("session_narrative", "")
    if not narrative or len(narrative) < 500:
        return {"original_chars": len(narrative), "compressed_chars": len(narrative), "compressed": False}

    prompt = (
        "Compress the following session narrative to 150 words or fewer. "
        "Preserve: user preferences, completed actions, pending tasks, key facts.\n\n"
        f"{narrative}"
    )
    response = genai.generate_content(
        model="gemini-2.0-flash-lite",  # Use lightest model for compression
        contents=[{"parts": [{"text": prompt}]}],
    )
    compressed = response.text.strip() if response and response.text else "[No summary]"
    tool_context.state["session_narrative"] = compressed
    return {"original_chars": len(narrative), "compressed_chars": len(compressed), "compressed": True}
```

## Edge Cases

- **Compression loop** (compress → still too large → compress again indefinitely):
  Set a maximum of 3 compression passes. After 3, truncate to absolute minimum.
- **Numbers and codes lost in LLM compression**: Explicitly list "preserve all
  booking references, prices, dates, and codes" in the summarization prompt.
- **UTF-8 multi-byte characters truncated mid-character**: Use Python's string
  slicing (which operates on code points, not bytes) for truncation.
- **Structural compaction removes the last relevant item**: Set a minimum floor
  of 3 items for list compaction, even if it exceeds the size target slightly.

## Integration Notes

- **`google-adk-memory-summarization`**: LLM-based compression for session history
  is formalized in that skill. This skill covers the general compression strategies.
- **`google-adk-context-window-management`**: Compression is applied to make
  memory fit within context window limits.
- **`google-adk-memory-eviction-strategies`**: Eviction removes entire entries;
  compression reduces their size. Apply compression before eviction.
- **`google-adk-working-memory`**: Compressed values are stored in short-term
  or long-term state and injected into working memory as needed.
