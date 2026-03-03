---
name: tool-result-parsing
description: >
  Use this skill when a tool returns raw, unstructured, or complex output that
  must be parsed, extracted, normalized, or reformatted before being injected
  into the LLM context or passed to the next tool in a chain.
compatibility:
  - Claude Code
  - Google ADK
  - OpenAI Agents
  - MCP-based agents
  - Tool-using LLM agents
metadata:
  author: agent-skills-generator
  version: "1.0"
---

## Purpose

This skill governs how to extract relevant data from tool results — particularly
when tools return large JSON payloads, HTML, raw text, or nested structures that
the LLM does not need in full. It defines extraction, normalization, and
summarization patterns that reduce token usage and prevent context overflow.

## When to Use

- When a tool returns more data than the LLM needs (large API payloads).
- When the output is HTML, XML, or raw text that must be cleaned.
- When downstream tools or the LLM needs a specific field from a complex result.
- When multiple tools return different schemas for the same logical entity.

## When NOT to Use

- Do not use when the tool already returns a clean, minimal dict.
- Do not use to validate the result schema — use `tool-validation` for that.
- Do not over-parse: only extract what is actually needed for the current task.

## Capabilities

- Extracts specific fields from nested JSON tool results using dot-path notation.
- Strips HTML tags and normalizes whitespace from text results.
- Truncates large results to a token-safe size with a summary prefix.
- Normalizes inconsistent API response schemas to a canonical format.
- Converts lists of raw records into structured summaries.

## Step-by-Step Instructions

1. **Identify what the consumer needs** — the LLM context or next tool. List
   the exact fields or data points required.

2. **Extract required fields** from the raw result using safe access:
   ```python
   title = result.get("data", {}).get("article", {}).get("title", "")
   ```
   Use `get()` with defaults — never assume a field exists.

3. **Normalize text content**:
   - If raw HTML: strip tags with `BeautifulSoup` or regex.
   - If Markdown: pass through as-is.
   - Remove duplicate whitespace and newlines.

4. **Truncate long text fields** to a safe limit (default: 2000 characters):
   ```python
   content = raw_content[:2000] + ("..." if len(raw_content) > 2000 else "")
   ```

5. **Build and return a canonical result dict** with only the required fields.

6. **Handle missing fields gracefully**: If a required field is absent, set it
   to `None` and note its absence in a `"warnings"` list.

7. **Log the before/after sizes** (token count or character count) for
   observability.

## Input Format

```
raw_result: dict | str | list    # Raw tool output
required_fields: list[str]       # Fields needed by LLM or next tool
max_text_length: int             # Default: 2000
```

## Output Format

```python
{
  "parsed": {
    "field_a": "value",
    "field_b": 42,
    "text_content": "Truncated text content..."
  },
  "warnings": ["field_c was missing from the raw result"],
  "original_size": 15420,    # characters before parsing
  "parsed_size": 320         # characters after parsing
}
```

## Error Handling

- **`raw_result` is `None`**: Return `{"error": "Tool returned no result.", "error_type": "unexpected"}`.
- **`raw_result` is a string instead of dict**: Attempt JSON parsing; if it
  fails, treat the whole string as `"text_content"`.
- **Required field is completely absent**: Add to `"warnings"`, do not raise.
- **HTML parsing fails**: Fall back to regex-based tag stripping.

## Examples

### Example 1 — Extracting from a deep JSON payload

```python
def parse_product_result(raw: dict, required_fields: list[str]) -> dict:
    """Extracts fields from a raw product API response."""
    parsed = {}
    warnings = []

    field_paths = {
        "name": ["product", "details", "name"],
        "price": ["product", "pricing", "amount"],
        "currency": ["product", "pricing", "currency"],
        "in_stock": ["product", "inventory", "available"],
    }

    for field in required_fields:
        path = field_paths.get(field)
        if not path:
            warnings.append(f"No path defined for field '{field}'")
            parsed[field] = None
            continue
        value = raw
        for key in path:
            value = value.get(key) if isinstance(value, dict) else None
            if value is None:
                break
        if value is None:
            warnings.append(f"Field '{field}' not found in result")
        parsed[field] = value

    return {"parsed": parsed, "warnings": warnings}
```

### Example 2 — Cleaning and truncating HTML web content

```python
import re

def parse_web_result(raw: dict, max_length: int = 2000) -> dict:
    """Cleans HTML from a web tool result and truncates to max_length."""
    html_content = raw.get("html", raw.get("content", ""))
    # Strip HTML tags
    text = re.sub(r"<[^>]+>", " ", html_content)
    # Normalize whitespace
    text = re.sub(r"\s+", " ", text).strip()
    # Truncate
    truncated = text[:max_length] + ("..." if len(text) > max_length else "")
    return {
        "parsed": {"text_content": truncated},
        "original_size": len(html_content),
        "parsed_size": len(truncated),
        "warnings": [] if len(text) <= max_length else ["Content truncated to 2000 chars"],
    }
```

### Example 3 — Normalizing inconsistent API schemas

```python
def normalize_user_result(raw: dict) -> dict:
    """Normalizes user objects from different API versions to canonical schema."""
    return {
        "id": raw.get("user_id") or raw.get("id") or raw.get("userId"),
        "name": raw.get("full_name") or raw.get("name") or f"{raw.get('first_name', '')} {raw.get('last_name', '')}".strip(),
        "email": raw.get("email") or raw.get("email_address"),
    }
```

## Edge Cases

- **Result is a list of items**: Parse each item individually using the same
  field extraction logic; return a `"parsed_items"` list.
- **Binary data in result** (e.g., PDF bytes): Do NOT inject into LLM context.
  Save to a file and return the file path instead.
- **Nested arrays with variable depth**: Flatten using recursion with a max
  depth of 3 to avoid infinite recursion.
- **Numbers returned as strings**: Use `int()` or `float()` to coerce. Log the
  coercion in `warnings`.

## Integration Notes

- **Google ADK**: Apply result parsing in `after_tool_callback` on `LlmAgent`
  to transform results before they are injected into model context.
- **tool-chaining skill**: Apply result parsing as the step between two tools
  to extract the exact fields needed by the next tool.
- **tool-validation skill**: Run post-call validation BEFORE parsing to ensure
  the result has the expected structure.
- **Context limits**: In Gemini 2.0 Flash, each tool result injected into
  context consumes tokens. Parsed/truncated results reduce costs significantly.
