---
name: structured-json-output
description: >
  Use this skill when configuring a Google ADK LlmAgent to produce validated,
  machine-parseable JSON output using LlmAgent.output_schema (Pydantic) and/or
  GenerateContentConfig.response_mime_type="application/json". Covers the complete
  structured output workflow: Pydantic schema design, applying output_schema on
  LlmAgent, storing output in session state via output_key, and parsing structured
  output in downstream agents and tools.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: model-configuration
---

## Purpose

This skill implements the complete structured JSON output workflow for Google ADK
`LlmAgent`. It combines `LlmAgent.output_schema` (ADK-level Pydantic validation)
with `GenerateContentConfig.response_mime_type="application/json"` (model-level
JSON constraint) to produce reliably parsed, typed, and validated JSON from an
ADK agent. Structured JSON output is the foundation for machine-to-machine
communication between agents in a pipeline, for API integration, and for any
workflow requiring typed data from LLM inference.

## When to Use

- When the agent's output is consumed programmatically by another system,
  agent, or tool (not displayed to a human user).
- When the agent must extract structured data from unstructured text (documents,
  search results, conversation history).
- When the downstream system requires typed, validated JSON with specific field
  names and types.
- When `response_match_score` is high-variance due to inconsistent output
  formatting — structured output provides the reliability fix.
- When building a multi-agent pipeline where sub-agents need to pass structured
  data to each other via `session.state`.

## When NOT to Use

- Do not use structured JSON output for user-facing chat responses — JSON is
  not human-friendly.
- Do not use `output_schema` simultaneously with `tools` on the same `LlmAgent`
  — ADK does not support function calling and structured output in a single turn.
  Use `SequentialAgent` to chain them.
- Do not use structured output for sub-agents in `ParallelAgent` that share state
  keys — concurrent writes to the same `output_key` cause race conditions.

## Google ADK Context

- **`LlmAgent.output_schema`**: Accepts a Pydantic `BaseModel` class. ADK
  instructs the model to generate JSON conforming to the schema, then parses
  and validates it automatically.
- **`LlmAgent.output_key`**: When `output_schema` is set, the validated output
  is stored in `session.state[output_key]`. Subsequent agents can access it via
  `{output_key?}` injection or `ReadonlyContext`.
- **`GenerateContentConfig.response_mime_type`**: Set to `"application/json"` to
  enforce JSON generation at model level, complementing `output_schema` validation.
- **`GenerateContentConfig.response_schema`**: Lower-level model-level constraint.
  Use `LlmAgent.output_schema` (ADK-idiomatic) instead when possible.
- **Tools + output_schema conflict**: ADK raises an error if both `tools` (non-empty)
  and `output_schema` are set on the same agent. Pattern: `SequentialAgent([tool_caller, schema_agent])`.
- **ADK auto-retry**: If the model generates invalid JSON that fails Pydantic
  validation, ADK retries up to 3 times automatically.
- **Output stored in state**: `output_key="my_output"` stores the validated JSON
  string in `session.state["my_output"]`. Downstream agents read it via
  `ctx.state.get("my_output")` or `{my_output?}` injection.

## Capabilities

- Enforces Pydantic-validated JSON output via `LlmAgent.output_schema`.
- Stores structured output in `session.state` via `output_key`.
- Combines `output_schema` with `response_mime_type="application/json"`.
- Chains tool-calling and structured extraction in a `SequentialAgent` pipeline.
- Parses and validates structured output in downstream agents and callbacks.
- Designs complex nested Pydantic schemas for multi-field extraction.
- Handles schema validation failures with retry and fallback strategies.

## Step-by-Step Instructions

1. **Design the Pydantic schema** with precise field types and descriptions:
   ```python
   from pydantic import BaseModel, Field
   from typing import Optional, List, Literal

   class ProductListing(BaseModel):
       name: str = Field(description="Product name")
       price_usd: float = Field(description="Price in USD")
       category: Literal["electronics", "clothing", "food", "other"]
       in_stock: bool
       tags: List[str] = Field(default_factory=list)
       description: Optional[str] = None
   ```

2. **Configure `GenerateContentConfig`** with `response_mime_type` and low sampling:
   ```python
   from google.genai.types import GenerateContentConfig
   config = GenerateContentConfig(
       temperature=0.1,
       top_k=20,
       top_p=0.90,
       max_output_tokens=2048,
       response_mime_type="application/json",
   )
   ```

3. **Define the extraction instruction** — describe what goes in each field:
   ```python
   INSTRUCTION = """
   Extract product listing data from the product_text in session state.
   Populate every field. For category, choose the closest match from the allowed values.
   For tags, extract key product attributes as short phrases.
   """
   ```

4. **Create the extraction agent**:
   ```python
   extractor = LlmAgent(
       name="product_extractor",
       model="gemini-2.0-flash",
       instruction=INSTRUCTION,
       generate_content_config=config,
       output_schema=ProductListing,
       output_key="product_listing",
   )
   ```

5. **Chain with a tool-calling agent** if raw data must be fetched first:
   ```python
   from google.adk.agents import SequentialAgent
   pipeline = SequentialAgent(
       name="product_pipeline",
       sub_agents=[raw_data_fetcher, extractor],
   )
   ```

6. **Read structured output in downstream agent/callback**:
   ```python
   import json
   from google.adk.agents.readonly_context import ReadonlyContext
   def use_output(ctx: ReadonlyContext) -> str:
       raw = ctx.state.get("product_listing")
       if raw:
           product = ProductListing.model_validate_json(raw if isinstance(raw, str) else json.dumps(raw))
           return f"Extracted: {product.name} at ${product.price_usd}"
       return "No product data available."
   ```

7. **Handle validation failure gracefully**:
   ```python
   from pydantic import ValidationError
   try:
       product = ProductListing.model_validate_json(raw_json)
   except ValidationError as exc:
       logger.error(f"Schema validation failed: {exc.error_count()} errors")
       product = None
   ```

## Input Format

```python
{
  "model": "gemini-2.0-flash",
  "temperature": 0.1,
  "top_k": 20,
  "top_p": 0.90,
  "max_tokens": 2048,
  "stop_sequences": [],
  "stream": False,
  "response_schema": {"type": "ProductListing"},
  "deterministic": False
}
```

## Output Format

```python
{
  "model_used": "gemini-2.0-flash",
  "configuration_applied": {
    "output_schema": "ProductListing",
    "response_mime_type": "application/json",
    "output_key": "product_listing"
  },
  "output_valid": True,
  "structured_output": {
    "name": "Wireless Headphones Pro",
    "price_usd": 149.99,
    "category": "electronics",
    "in_stock": True,
    "tags": ["wireless", "noise-cancelling", "bluetooth"],
    "description": "Premium noise-cancelling wireless headphones"
  }
}
```

## Error Handling

- **Pydantic `ValidationError`**: ADK retries up to 3 times. If all fail, the
  agent raises an exception. Add `after_agent_callback` to catch and log.
- **Partial JSON in output** (max_output_tokens too small): Increase the token
  limit. For large schemas, set `max_output_tokens=4096`.
- **Required field missing**: Either make the field `Optional` with a default,
  or add explicit instruction: "Always populate the `[field_name]` field."
- **`output_key` collision** (two agents writing to the same key in `ParallelAgent`):
  Use unique `output_key` names per sub-agent in parallel pipelines.
- **Type mismatch** (e.g., price returned as `"149.99"` instead of `149.99`):
  Add explicit type instructions: "Return price_usd as a number (not a string)."
- **Enum field violates constraint**: Add explicit enum values in the instruction:
  "For category, use exactly one of: electronics, clothing, food, other."

## Examples

### Example 1 — Complete structured extraction pipeline

```python
from google.adk.agents import LlmAgent, SequentialAgent
from google.genai.types import GenerateContentConfig
from pydantic import BaseModel, Field
from typing import Optional, List, Literal
import json

class FlightAnalysis(BaseModel):
    total_flights: int = Field(description="Number of flights found")
    price_range_usd: dict = Field(description="{'min': float, 'max': float}")
    cheapest_airline: str = Field(description="Airline of the cheapest flight")
    fastest_duration_minutes: int = Field(description="Duration of fastest flight in minutes")
    recommended_flight_id: str = Field(description="ID of the recommended flight")
    recommendation_reason: str = Field(description="Brief reason for recommendation (max 50 words)")
    has_direct_flights: bool = Field(description="Whether any direct (non-stop) flights exist")
    tags: List[str] = Field(description="Key characteristics of available flights", default_factory=list)

EXTRACTION_CONFIG = GenerateContentConfig(
    temperature=0.05,
    top_k=10,
    top_p=0.85,
    max_output_tokens=2048,
    response_mime_type="application/json",
)

ANALYSIS_INSTRUCTION = """
Analyze the raw_search_results in session state and extract structured flight analysis.
Populate ALL fields from the actual search data.
For price_range_usd, provide {"min": <lowest_price>, "max": <highest_price>}.
For tags, include characteristics like "direct", "overnight", "budget", "premium" if applicable.
Limit recommendation_reason to 50 words maximum.
"""

# Tool-calling agent (separate from output_schema agent)
search_agent = LlmAgent(
    name="flight_searcher",
    model="gemini-2.0-flash",
    instruction="Search for flights and store results in state key 'raw_search_results'.",
    tools=[search_flights],
    output_key="raw_search_results",
)

# Schema-constrained extraction agent (no tools)
analysis_agent = LlmAgent(
    name="flight_analyzer",
    model="gemini-2.0-flash",
    instruction=ANALYSIS_INSTRUCTION,
    generate_content_config=EXTRACTION_CONFIG,
    output_schema=FlightAnalysis,
    output_key="flight_analysis",
)

pipeline = SequentialAgent(
    name="flight_analysis_pipeline",
    sub_agents=[search_agent, analysis_agent],
)
```

### Example 2 — Reading and using structured output in next agent

```python
from google.adk.agents import LlmAgent
from google.adk.agents.readonly_context import ReadonlyContext
import json

def booking_instruction_provider(ctx: ReadonlyContext) -> str:
    """Injects structured flight analysis into the booking agent's instruction."""
    raw = ctx.state.get("flight_analysis")
    if not raw:
        return "Help the user book a flight. No analysis data is available yet."
    try:
        analysis = FlightAnalysis.model_validate_json(
            raw if isinstance(raw, str) else json.dumps(raw)
        )
        return f"""
You are a booking agent. Based on the analysis:
- Recommended flight: {analysis.recommended_flight_id}
- Reason: {analysis.recommendation_reason}
- Price range: ${analysis.price_range_usd.get('min')}–${analysis.price_range_usd.get('max')}

Present this recommendation to the user and confirm before booking.
"""
    except Exception:
        return "Help the user book a flight."

booking_agent = LlmAgent(
    name="booking_agent",
    model="gemini-2.0-flash",
    instruction=booking_instruction_provider,
    tools=[process_payment],
)
```

### Example 3 — Schema validation in after_agent_callback

```python
import json
import logging
from typing import Optional
from pydantic import ValidationError
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content, Part

logger = logging.getLogger("schema_validator")

def validate_structured_output(callback_context: CallbackContext, response: Optional[Content]) -> Optional[Content]:
    """Validates the structured output and provides a fallback on schema failure."""
    output_key = "flight_analysis"
    raw = callback_context.state.get(output_key)
    if not raw:
        return response

    try:
        _ = FlightAnalysis.model_validate_json(
            raw if isinstance(raw, str) else json.dumps(raw)
        )
        logger.info(f"Schema validation passed for {output_key}.")
    except ValidationError as exc:
        logger.error(f"Schema validation failed: {exc.error_count()} errors. Key: {output_key}")
        # Return error message to caller
        return Content(parts=[Part(text=f"Output validation failed: {exc.error_count()} schema errors.")])
    return response
```

## Edge Cases

- **Score inconsistency across retries**: ADK retries generate slightly different
  JSON on each attempt. All 3 retries may fail for complex schemas. Simplify
  the schema or split into multiple extraction agents.
- **Null vs. missing fields in JSON**: `Optional[T]` handles null. Missing fields
  (the key doesn't exist in the JSON) are also handled by Pydantic if the field
  has a default.
- **`output_key` stores a dict, not a JSON string**: ADK may store the parsed
  Pydantic object as a dict in state. Use `json.dumps(raw)` before calling
  `model_validate_json` if `isinstance(raw, dict)`.
- **Schema changes break stored state**: If the schema changes and old state
  exists, `model_validate_json` fails. Add schema version to `output_key`:
  `output_key="flight_analysis_v2"`.

## Integration Notes

- **`google-adk-structured-prompting`**: Covers instruction design for structured
  output agents — the content guidance for each field.
- **`google-adk-response-schemas`**: Covers `GenerateContentConfig.response_schema`
  and `response_mime_type` — the lower-level model configuration for the same goal.
- **`google-adk-context-injection`**: Downstream agents access structured output
  via `{output_key?}` injection into their instruction.
- **`SequentialAgent`**: The required pattern for tool-calling + structured output.
  Tool caller comes first; extraction agent with `output_schema` comes second.
- **`google-adk-output-formatting`**: For user-facing formatting of the structured
  data after extraction, a separate presentation agent formats the output for display.
