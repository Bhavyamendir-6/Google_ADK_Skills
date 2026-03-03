---
name: web-tools
description: >
  Use this skill when building agent tools that fetch web pages, extract content
  from HTML, perform web searches, or interact with browser-rendered content.
  Covers HTTP fetching, HTML parsing, search API integration, and content
  extraction patterns.
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

This skill defines patterns for building web-facing agent tools: fetching URLs,
parsing HTML, extracting structured data, and querying search engines. It
distinguishes between static HTTP fetches and JavaScript-rendered content
requiring a headless browser, and provides the correct approach for each case.

## When to Use

- When an agent needs to read content from a specific URL.
- When implementing web search functionality as a tool.
- When extracting structured data from HTML pages (tables, articles, listings).
- When the agent needs to check whether a URL is reachable or returns content.

## When NOT to Use

- Do not use for calling REST APIs with JSON responses — use `external-api-tools`.
- Do not use for database queries — use `database-tools`.
- Do not use for authenticated browser sessions requiring login flows without
  a headless browser strategy.

## Capabilities

- Fetches static web pages with `requests` and parses HTML with `BeautifulSoup`.
- Extracts article text, tables, and metadata from parsed HTML.
- Integrates with web search APIs (Google Custom Search, SerpAPI, Brave Search).
- Handles JavaScript-rendered pages with `playwright` (async).
- Respects `robots.txt` and implements rate-limiting between fetches.
- Returns clean, truncated text safe for LLM context injection.

## Step-by-Step Instructions

1. **Determine if the page is static or JS-rendered**:
   - Static: Use `requests` + `BeautifulSoup`.
   - JS-rendered: Use `playwright` with a headless Chromium browser.

2. **For static pages — fetch and parse**:
   ```python
   import requests
   from bs4 import BeautifulSoup
   response = requests.get(url, timeout=10, headers={"User-Agent": "Mozilla/5.0"})
   response.raise_for_status()
   soup = BeautifulSoup(response.text, "html.parser")
   ```

3. **Extract the main content**:
   - Remove `<script>`, `<style>`, `<nav>`, `<footer>`, `<header>` tags.
   - Extract the main `<article>` or `<main>` element if present.
   - Fall back to extracting all `<p>` tag text.

4. **Clean the text**:
   - Strip remaining HTML tags.
   - Normalize whitespace.
   - Truncate to 3000 characters.

5. **For web search** — use a search API, not direct scraping:
   ```python
   def search_web(query: str, num_results: int = 5) -> dict:
       response = requests.get(
           "https://www.googleapis.com/customsearch/v1",
           params={"q": query, "num": num_results, "key": API_KEY, "cx": SEARCH_ENGINE_ID},
           timeout=10,
       )
       ...
   ```

6. **Apply `@tool_error_handler`** for HTTP errors, timeouts, and parse failures.

7. **Return a normalized result** with `url`, `title`, `content`, `fetched_at`.

## Input Format

For `fetch_page`:
```
url: str              # Full URL including https://
max_content_length: int  # Default: 3000 characters
```

For `search_web`:
```
query: str            # Search query string
num_results: int      # Default: 5, max: 10
```

## Output Format

`fetch_page` result:
```python
{
  "url": "https://example.com/article",
  "title": "Example Article Title",
  "content": "First 3000 characters of the main article text...",
  "truncated": True,
  "fetched_at": "2024-01-15T10:30:00Z"
}
```

`search_web` result:
```python
{
  "query": "python async programming",
  "results": [
    {"title": "Async IO in Python", "url": "https://...", "snippet": "..."},
  ],
  "count": 5
}
```

## Error Handling

- HTTP 403 (blocked) → `{"error": "Access denied by website.", "error_type": "permanent", "recoverable": False}`
- HTTP 404 → `{"error": "Page not found.", "error_type": "not_found", "recoverable": False}`
- HTTP 5xx → `{"error": "Server error.", "error_type": "transient", "recoverable": True}`
- `requests.Timeout` → `{"error": "Request timed out.", "error_type": "transient", "recoverable": True}`
- Empty content after parsing → `{"error": "No readable content found at URL.", "error_type": "not_found", "recoverable": False}`
- Invalid URL format → `{"error": "Invalid URL format.", "error_type": "invalid_input", "recoverable": False}`

## Examples

### Example 1 — Static page fetch tool

```python
import requests
from bs4 import BeautifulSoup
from datetime import datetime, timezone

@tool_error_handler
def fetch_page(url: str, max_content_length: int = 3000) -> dict:
    """Fetches and extracts the main text content from a web page URL.

    Args:
        url (str): Full URL of the page to fetch, e.g. 'https://example.com/article'.
        max_content_length (int): Maximum characters of content to return. Defaults to 3000.

    Returns:
        dict: Contains 'url', 'title', 'content', 'truncated', and 'fetched_at'.
    """
    headers = {"User-Agent": "Mozilla/5.0 (compatible; AgentBot/1.0)"}
    response = requests.get(url, timeout=10, headers=headers)
    response.raise_for_status()

    soup = BeautifulSoup(response.text, "html.parser")
    for tag in soup(["script", "style", "nav", "footer", "header", "aside"]):
        tag.decompose()

    title = soup.title.string.strip() if soup.title else ""
    main = soup.find("article") or soup.find("main") or soup.body
    text = " ".join(p.get_text() for p in main.find_all("p")) if main else ""
    text = " ".join(text.split())  # normalize whitespace

    truncated = len(text) > max_content_length
    content = text[:max_content_length] + ("..." if truncated else "")

    return {
        "url": url,
        "title": title,
        "content": content,
        "truncated": truncated,
        "fetched_at": datetime.now(timezone.utc).isoformat(),
    }
```

### Example 2 — Web search tool with SerpAPI

```python
import os

@tool_error_handler
def search_web(query: str, num_results: int = 5) -> dict:
    """Searches the web and returns top result titles, URLs, and snippets.

    Args:
        query (str): The search query string.
        num_results (int): Number of results to return. Defaults to 5.

    Returns:
        dict: Contains 'query', 'results' (list of title/url/snippet), and 'count'.
    """
    num_results = min(num_results, 10)
    response = requests.get(
        "https://serpapi.com/search",
        params={"q": query, "num": num_results, "api_key": os.environ["SERPAPI_KEY"]},
        timeout=15,
    )
    response.raise_for_status()
    data = response.json()

    results = [
        {"title": r.get("title"), "url": r.get("link"), "snippet": r.get("snippet")}
        for r in data.get("organic_results", [])[:num_results]
    ]
    return {"query": query, "results": results, "count": len(results)}
```

### Example 3 — Playwright JS-rendered page (async)

```python
from playwright.async_api import async_playwright

async def fetch_js_page(url: str) -> dict:
    """Fetches content from a JavaScript-rendered page using headless browser."""
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        await page.goto(url, timeout=30000)
        await page.wait_for_load_state("networkidle")
        content = await page.inner_text("body")
        title = await page.title()
        await browser.close()
    content = " ".join(content.split())[:3000]
    return {"url": url, "title": title, "content": content}
```

## Edge Cases

- **Paywalled content**: Fetch returns login wall HTML. Detect by checking if
  content length < 500 and return `not_found` error.
- **PDF URLs**: Detect `Content-Type: application/pdf`. Return error directing
  to a PDF-extraction tool.
- **Redirects**: `requests` follows redirects automatically. Log the final URL.
- **Non-UTF-8 encoding**: Use `response.apparent_encoding` to detect and decode
  correctly.
- **Rate limiting by the target site**: Add a configurable `delay_seconds`
  parameter for tools that bulk-fetch multiple URLs.

## Integration Notes

- **Google ADK**: Register `fetch_page` and `search_web` directly as tools.
  For Gemini models, also consider the built-in `google_search` tool from ADK.
- **Google Search Grounding**: For Gemini agents, prefer the native Google
  Search grounding feature over custom search tools for better accuracy.
- **MCP**: Use the `fetch` MCP server for simple URL fetching in MCP-based
  deployments.
- **tool-result-parsing skill**: Always apply text cleaning and truncation
  (steps 3–4) before returning web content to the LLM.
