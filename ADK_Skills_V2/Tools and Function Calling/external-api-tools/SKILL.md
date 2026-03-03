---
name: external-api-tools
description: >
  Use this skill when building a tool that wraps an external REST or GraphQL
  API. Covers authentication, request construction, response handling, rate
  limiting, and exposing the API capabilities as LLM-callable tool functions.
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

This skill defines the standard pattern for wrapping external HTTP APIs as
agent tools. It ensures that authentication credentials are managed securely,
requests are constructed correctly, responses are parsed and normalized, and
failures are handled consistently — making the API reliably callable by an LLM.

## When to Use

- When building a tool that calls a third-party REST API (e.g., weather, maps,
  payments, CRM, data services).
- When adding HTTP-based integrations to an agent's toolset.
- When an existing API client needs to be adapted to the agent tool interface.

## When NOT to Use

- Do not use for database tools (see `database-tools`).
- Do not use for browser/web scraping tools (see `web-tools`).
- Do not use for MCP tool servers — those have their own setup patterns.

## Capabilities

- Implements API authentication: Bearer tokens, API keys, OAuth2.
- Constructs typed HTTP requests with correct headers and parameters.
- Parses JSON responses into normalized dicts.
- Handles HTTP error codes with structured error results.
- Manages rate limits with retry support.
- Keeps credentials out of tool function signatures.

## Step-by-Step Instructions

1. **Create an API client class** that encapsulates session setup,
   authentication, and base URL configuration:
   ```python
   class WeatherApiClient:
       BASE_URL = "https://api.weather.example.com/v1"
       def __init__(self, api_key: str):
           self.session = requests.Session()
           self.session.headers.update({"Authorization": f"Bearer {api_key}"})
   ```

2. **Load credentials from environment variables** — NEVER hardcode them:
   ```python
   import os
   api_key = os.environ["WEATHER_API_KEY"]
   ```

3. **Define a method per API endpoint** on the client class:
   ```python
   def get_current_weather(self, city: str, units: str = "metric") -> dict:
       response = self.session.get(
           f"{self.BASE_URL}/current",
           params={"city": city, "units": units},
           timeout=10,
       )
       response.raise_for_status()
       return response.json()
   ```

4. **Create a tool wrapper function** (not a method) that:
   - Accepts only LLM-friendly primitive parameters.
   - Instantiates or uses a module-level cached client.
   - Calls the client method.
   - Returns a normalized result dict.
   - Applies `@tool_error_handler` and `@with_retries`.

5. **Normalize the response** to remove API-specific fields the LLM does not
   need (see `tool-result-parsing`).

6. **Handle HTTP status codes explicitly**:
   - 200: Parse and return result.
   - 401/403: Return permanent auth error.
   - 429: Return transient rate-limit error.
   - 404: Return not-found error.
   - 5xx: Return transient server error.

7. **Register the wrapper function** with the agent (see `tool-registration`).

## Input Format

```
api_name: <str>
base_url: <str>
auth_method: bearer_token | api_key_header | api_key_query | oauth2
endpoints:
  - method: GET | POST | PUT | DELETE
    path: <str>
    params: dict
    body_schema: dict | None
credential_env_var: <str>
```

## Output Format

Normalized tool result dict:
```python
{
  "city": "London",
  "temperature_celsius": 12.5,
  "condition": "Cloudy",
  "humidity_percent": 78
}
```

Error result (on failure):
```python
{
  "error": "Weather API returned 429: rate limit exceeded.",
  "error_type": "transient",
  "recoverable": True
}
```

## Error Handling

- HTTP 401/403 → `{"error": "Authentication failed.", "error_type": "permanent", "recoverable": False}`
- HTTP 429 → `{"error": "Rate limit exceeded.", "error_type": "transient", "recoverable": True}` (include `Retry-After` header value as `"retry_after_seconds"`)
- HTTP 404 → `{"error": "Resource not found.", "error_type": "not_found", "recoverable": False}`
- HTTP 5xx → `{"error": "Server error.", "error_type": "transient", "recoverable": True}`
- `requests.Timeout` → `{"error": "Request timed out.", "error_type": "transient", "recoverable": True}`
- JSON decode error → `{"error": "Invalid JSON response.", "error_type": "unexpected", "recoverable": False}`

## Examples

### Example 1 — Complete external API tool

```python
import os
import requests
from functools import lru_cache

@lru_cache(maxsize=1)
def _get_weather_client() -> "WeatherApiClient":
    return WeatherApiClient(api_key=os.environ["WEATHER_API_KEY"])

class WeatherApiClient:
    BASE_URL = "https://api.openweathermap.org/data/2.5"

    def __init__(self, api_key: str):
        self.session = requests.Session()
        self.api_key = api_key

    def get_current(self, city: str, units: str = "metric") -> dict:
        resp = self.session.get(
            f"{self.BASE_URL}/weather",
            params={"q": city, "units": units, "appid": self.api_key},
            timeout=10,
        )
        resp.raise_for_status()
        return resp.json()

@tool_error_handler
@with_retries(max_attempts=3, base_delay=1.0)
def get_weather(city: str, units: str = "metric") -> dict:
    """Get current weather conditions for a city.

    Args:
        city (str): City name, e.g. 'London'.
        units (str): 'metric' for Celsius, 'imperial' for Fahrenheit. Defaults to 'metric'.

    Returns:
        dict: Contains 'city', 'temperature', 'condition', 'humidity_percent'.
    """
    client = _get_weather_client()
    raw = client.get_current(city=city, units=units)
    return {
        "city": raw["name"],
        "temperature": raw["main"]["temp"],
        "condition": raw["weather"][0]["description"],
        "humidity_percent": raw["main"]["humidity"],
    }
```

### Example 2 — POST request tool

```python
@tool_error_handler
def create_support_ticket(title: str, description: str, priority: str = "medium") -> dict:
    """Creates a new support ticket in the helpdesk system.

    Args:
        title (str): Short title of the issue.
        description (str): Detailed description.
        priority (str): 'low', 'medium', or 'high'. Defaults to 'medium'.

    Returns:
        dict: Contains 'ticket_id' and 'url'.
    """
    resp = requests.post(
        "https://api.helpdesk.example.com/tickets",
        json={"title": title, "description": description, "priority": priority},
        headers={"Authorization": f"Bearer {os.environ['HELPDESK_API_KEY']}"},
        timeout=15,
    )
    resp.raise_for_status()
    data = resp.json()
    return {"ticket_id": data["id"], "url": data["links"]["self"]}
```

## Edge Cases

- **API uses non-standard auth** (e.g., HMAC signature): Implement signing
  logic in the client class `__init__` or as a `requests.AuthBase` subclass.
- **Paginated responses**: Implement a `_paginate()` helper that fetches all
  pages and returns a combined list. Expose `max_results` as a tool parameter.
- **Large payloads (>50KB)**: Download to a temp file and return the file path
  instead of the raw content.
- **API returns XML**: Parse with `xml.etree.ElementTree` and convert to dict
  before returning.

## Integration Notes

- **Google ADK**: Register the wrapper function (not the client method) in
  `LlmAgent(tools=[get_weather])`.
- **MCP**: Wrap the same client method with the MCP `@server.tool()` decorator
  for MCP-based deployments.
- **Credentials security**: Store API keys in Google Secret Manager or
  environment variables. Never log or return credentials.
- **tool-retries skill**: Apply `@with_retries` on tools calling APIs
  with known rate limits or intermittent availability.
