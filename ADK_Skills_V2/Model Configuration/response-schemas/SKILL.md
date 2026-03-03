---
name: response-schemas
description: >
  Use this skill when configuring response_mime_type and response_schema in a
  Google ADK LlmAgent's GenerateContentConfig to enforce structured output
  format at the model inference level. Covers setting response_mime_type to
  application/json, passing response_schema as a Pydantic model or JSON schema
  dict, and validating schema-conformant model output in ADK pipelines.
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

This skill configures `response_schema` and `response_mime_type` in Google ADK
`LlmAgent`'s `GenerateContentConfig`. Unlike `LlmAgent.output_schema` (which is
an ADK-level Pydantic schema applied after generation), `response_schema` in
`GenerateContentConfig` is a model-level constraint — it instructs the Gemini
model to produce output that natively conforms to the schema during generation.
This provides more reliable schema adherence than text-only formatting instructions,
and is the preferred approach when the model must output constrained JSON without
ADK-level validation overhead.

## When to Use

- When the model's output must conform to a JSON schema at generation time (not
  just validated after generation).
- When `LlmAgent.output_schema` validation failures are too frequent and a
  model-level constraint is needed.
- When building extraction agents that emit strictly typed JSON for downstream
  consumption.
- When integrating with external systems that require a specific JSON format.
- When `response_mime_type="application/json"` is needed to guarantee valid JSON output.

## When NOT to Use

- Do not apply `response_schema` to conversational agents that interact with
  human users — JSON output is not user-friendly.
- Do not use `response_schema` in `GenerateContentConfig` simultaneously with
  `LlmAgent.output_schema` — this creates a dual-schema conflict. Use one or
  the other.
- Do not use `response_schema` for agents that also call tools — ADK restricts
  function calling and constrained output to separate agent turns.

## Google ADK Context

- **`GenerateContentConfig.response_mime_type`**: Set to `"application/json"`
  to instruct the model to output valid JSON.
- **`GenerateContentConfig.response_schema`**: A JSON schema or Pydantic model
  that constrains the model's JSON output. Applied at the model inference level.
- **`LlmAgent.output_schema` vs. `GenerateContentConfig.response_schema`**:
  - `LlmAgent.output_schema`: ADK post-processes and validates the output (Pydantic).
  - `GenerateContentConfig.response_schema`: Constrains output at model level.
  - In practice, `LlmAgent.output_schema` is the preferred ADK-idiomatic approach.
    `response_schema` in `GenerateContentConfig` is lower-level.
- **Tool calls + response_schema conflict**: The Gemini API does not support
  simultaneous function calling and response_schema constraining in a single request.
  Use `SequentialAgent`: a tool-calling agent followed by a schema-constrained
  extraction agent.
- **Schema format**: `response_schema` accepts a Pydantic `BaseModel` class or
  a JSON schema dict.

## Capabilities

- Sets `response_mime_type="application/json"` for guaranteed JSON output.
- Applies `response_schema` at model inference level for native schema compliance.
- Reduces schema validation failure rates vs. text-only format instructions.
- Defines complex nested schemas for structured extraction at inference time.
- Validates model output against the schema via Pydantic post-processing.
- Supports both Pydantic BaseModel and JSON schema dict as schema formats.

## Step-by-Step Instructions

1. **Define the JSON schema** as a Pydantic model or JSON schema dict:
   ```python
   from pydantic import BaseModel
   from typing import Optional, List

   class FlightOption(BaseModel):
       airline: str
       flight_number: str
       departure_time: str
       arrival_time: str
       price_usd: float
       duration_minutes: int

   class FlightSearchOutput(BaseModel):
       flights: List[FlightOption]
       search_query: str
       total_found: int
   ```

2. **Configure `GenerateContentConfig`** with `response_schema` and `response_mime_type`:
   ```python
   from google.genai.types import GenerateContentConfig
   config = GenerateContentConfig(
       temperature=0.1,
       response_mime_type="application/json",
       response_schema=FlightSearchOutput,
   )
   ```

3. **Apply to the extraction agent** (NO tools — schema-constrained agents
   cannot call tools in the same request):
   ```python
   extractor = LlmAgent(
       name="flight_extractor",
       model="gemini-2.0-flash",
       instruction="Extract flight data from the search results in session state.",
       generate_content_config=config,
   )
   ```

4. **Chain with a tool-calling agent** using `SequentialAgent`:
   ```python
   pipeline = SequentialAgent(
       name="flight_pipeline",
       sub_agents=[tool_caller_agent, extractor],
   )
   ```

5. **Read and validate the output** from session state or the final response event:
   ```python
   import json
   for event in runner.run(...):
       if event.is_final_response() and event.content and event.content.parts:
           raw_json = event.content.parts[0].text
           result = FlightSearchOutput.model_validate_json(raw_json)
   ```

6. **Handle validation failures** with try/except around `model_validate_json`.

## Input Format

```python
{
  "model": "gemini-2.0-flash",
  "temperature": 0.1,
  "top_k": 20,
  "top_p": 0.90,
  "max_tokens": 1024,
  "stop_sequences": [],
  "stream": False,
  "response_schema": {
    "type": "object",
    "properties": {
      "airline": {"type": "string"},
      "price_usd": {"type": "number"}
    },
    "required": ["airline", "price_usd"]
  },
  "deterministic": False
}
```

## Output Format

```python
{
  "model_used": "gemini-2.0-flash",
  "configuration_applied": {
    "response_mime_type": "application/json",
    "response_schema": "FlightSearchOutput"
  },
  "output_valid": True,
  "structured_output": {
    "flights": [{"airline": "British Airways", "price_usd": 520.0, "...": "..."}],
    "total_found": 3
  }
}
```

## Error Handling

- **Schema validation failure** (model generates structurally non-conformant JSON):
  `model_validate_json` raises `ValidationError`. Retry once; if issue persists,
  lower temperature and top_k in `GenerateContentConfig`.
- **`response_schema` + tools conflict**: If the agent has tools in its tool list
  and `response_schema` is set in `GenerateContentConfig`, the model may refuse
  to generate function calls. Remove tools from the schema-constrained extraction agent.
- **JSON syntax error in model output**: Even with `response_mime_type="application/json"`,
  extremely long or complex schemas can produce malformed JSON. Add `max_output_tokens`
  to ensure the full schema object is generated before the context limit.
- **Enum field not respected**: Model may generate a value outside the Pydantic
  `Literal` or `Enum`. Add instruction text listing valid enum values.
- **Nested schema too complex**: Break into multiple extraction agents, each
  responsible for a sub-schema section.

## Examples

### Example 1 — Schema-constrained extraction pipeline

```python
from google.adk.agents import LlmAgent, SequentialAgent
from google.genai.types import GenerateContentConfig
from pydantic import BaseModel, Field
from typing import List, Optional

class FlightOption(BaseModel):
    airline: str = Field(description="Airline name")
    flight_number: str = Field(description="Flight number, e.g. BA178")
    departure_time: str = Field(description="Departure time in HH:MM format")
    arrival_time: str = Field(description="Arrival time in HH:MM format")
    price_usd: float = Field(description="Price in USD")
    duration_minutes: int = Field(description="Flight duration in minutes")

class FlightSearchOutput(BaseModel):
    flights: List[FlightOption] = Field(description="List of available flights")
    search_query: str = Field(description="The search query used")
    total_found: int = Field(description="Total flights found")
    best_value: Optional[str] = Field(description="Flight number of the best value option")

EXTRACTION_CONFIG = GenerateContentConfig(
    temperature=0.05,
    top_k=10,
    top_p=0.85,
    max_output_tokens=2048,
    response_mime_type="application/json",
    response_schema=FlightSearchOutput,
)

EXTRACTION_INSTRUCTION = """
Extract flight data from the raw_search_results in session state.
Populate all fields from the search results accurately.
For best_value, choose the flight with the best price-to-duration ratio.
"""

toll_caller = LlmAgent(name="searcher", model="gemini-2.0-flash",
                        instruction="Search for flights.", tools=[search_flights])

extractor = LlmAgent(
    name="flight_extractor",
    model="gemini-2.0-flash",
    instruction=EXTRACTION_INSTRUCTION,
    generate_content_config=EXTRACTION_CONFIG,
    output_key="flight_results",
)

pipeline = SequentialAgent(name="flight_pipeline", sub_agents=[toll_caller, extractor])
```

### Example 2 — JSON dict schema (without Pydantic)

```python
from google.genai.types import GenerateContentConfig

PRODUCT_SCHEMA = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "price": {"type": "number"},
        "category": {"type": "string", "enum": ["electronics", "clothing", "food"]},
        "in_stock": {"type": "boolean"},
    },
    "required": ["name", "price", "category", "in_stock"],
}

config = GenerateContentConfig(
    temperature=0.1,
    response_mime_type="application/json",
    response_schema=PRODUCT_SCHEMA,
)
```

### Example 3 — Validating schema output from event stream

```python
import json
from pydantic import ValidationError
from google.adk.runners import Runner
from google.genai.types import Content, Part

def run_and_validate(runner: Runner, user_id: str, session_id: str, query: str) -> FlightSearchOutput | None:
    """Runs extraction agent and validates schema output."""
    msg = Content(parts=[Part(text=query)])
    raw_json = ""
    for event in runner.run(user_id=user_id, session_id=session_id, new_message=msg):
        if event.is_final_response() and event.content and event.content.parts:
            raw_json = event.content.parts[0].text or ""
    try:
        return FlightSearchOutput.model_validate_json(raw_json)
    except (ValidationError, json.JSONDecodeError) as exc:
        import logging
        logging.getLogger("schema").error(f"Schema validation failed: {exc}. Raw: {raw_json[:200]}")
        return None
```

## Edge Cases

- **Schema too large to fit in response**: Increase `max_output_tokens`. A schema
  with 20+ fields may need 4000+ output tokens.
- **Nullable fields cause ADK Pydantic issues**: Use `Optional[type]` in Pydantic.
  Nullable JSON fields map to `Optional` in Python.
- **Streaming with response_schema**: The stream delivers partial JSON. Never
  parse partial JSON. Always wait for `is_final_response()`.
- **Array field with variable length**: The model may generate fewer array items
  than expected. Use `List[T]` with a minimum-count validation if needed.

## Integration Notes

- **`LlmAgent.output_schema`**: Preferred ADK-idiomatic approach. Use `output_schema`
  on the agent instead of `response_schema` in `GenerateContentConfig` for most cases.
- **`google-adk-structured-json-output`**: Covers the complete structured JSON
  output workflow including `output_schema`, `output_key`, and downstream parsing.
- **`google-adk-structured-prompting`**: Covers the instruction design for
  structured output agents.
- **`SequentialAgent`**: Required pattern for tool-calling + response_schema
  pipelines, since tools and response_schema cannot coexist in one agent request.
