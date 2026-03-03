---
name: structured-prompting
description: >
  Use this skill when designing prompts for Google ADK LlmAgents that require
  the output to follow a strict, machine-parseable structure. Covers using
  LlmAgent.output_schema with Pydantic models to enforce JSON output, combining
  structured schema with instruction prompts, and validating structured outputs
  from ADK agent responses programmatically.
compatibility:
  - claude-code
  - google-adk
  - mcp
  - openai-agents
metadata:
  author: google-adk-skills
  version: "1.0"
  framework: google-adk
  category: prompt-engineering
---

## Purpose

This skill implements structured output prompting for Google ADK `LlmAgent`.
When downstream systems, orchestrator agents, or tool calls require the agent's
output to be in a machine-parseable format (JSON, Pydantic model), structured
prompting defines the schema, instructs the model to conform to it, and validates
the output at runtime. ADK's `LlmAgent.output_schema` is the primary mechanism
for enforcing structured output — the model is constrained to generate a valid
JSON object matching the Pydantic schema.

## When to Use

- When the agent's output is consumed programmatically by another system or agent.
- When the output must conform to a strict schema (e.g., API response, database
  insert, downstream tool input).
- When text-based output formatting instructions alone produce inconsistent results.
- When a parent orchestrator agent needs to parse a sub-agent's output as
  structured data.
- When implementing extraction agents that must return typed, validated data.

## When NOT to Use

- Do not use `output_schema` for conversational agents that interact directly
  with human users — it forces JSON output, which is not user-friendly.
- Do not use structured output when the schema is so complex it inflates token
  count excessively.
- Do not combine `output_schema` with `tools` in the same `LlmAgent` — ADK
  does not support simultaneous tool calling and structured output in a single
  agent turn.

## Google ADK Context

- **`LlmAgent.output_schema`**: Accepts a Pydantic `BaseModel` class. ADK
  instructs the LLM to generate a valid JSON object matching the schema.
  The output is automatically parsed and validated.
- **`output_key`**: When `output_schema` is set, ADK stores the parsed output
  in `session.state[output_key]` automatically. Downstream agents or tools can
  read it from state.
- **`LlmAgent.instruction`**: Even with `output_schema`, the instruction must
  tell the model WHAT content to put in each field. The schema defines structure;
  the instruction defines content.
- **Tool calls vs. structured output**: ADK restricts `output_schema` to agents
  without tools. Use a separate extraction `LlmAgent` (no tools) to produce
  structured output from a tool-calling agent's raw output.
- **Pydantic validation**: ADK validates the model's JSON against the Pydantic
  schema. Invalid output raises a validation error and retries (up to 3 times).
- **`SequentialAgent`**: Use to chain a tool-calling agent followed by a
  structured-output extraction agent.

## Capabilities

- Enforces machine-parseable output via `LlmAgent.output_schema`.
- Stores structured output in `session.state[output_key]` for downstream access.
- Defines Pydantic schemas for complex nested output structures.
- Combines structured output with chain-of-thought reasoning fields.
- Validates structured output at runtime via Pydantic.
- Implements extraction pipelines: tool-calling agent → structured output agent.
- Injects structured output into downstream agents via state.

## Step-by-Step Instructions

1. **Define the output schema** as a Pydantic `BaseModel`:
   ```python
   from pydantic import BaseModel, Field
   from typing import Optional, List

   class FlightSearchResult(BaseModel):
       flights_found: int = Field(description="Number of flights found")
       cheapest_flight: Optional[str] = Field(description="Description of cheapest option")
       fastest_flight: Optional[str] = Field(description="Description of fastest option")
       recommendation: str = Field(description="Agent's recommendation")
   ```

2. **Write the instruction** describing what to put in each field:
   ```python
   instruction = """
   Analyze the flight search results and populate the output schema:
   - flights_found: Count of available flights from search_results
   - cheapest_flight: Brief description of the lowest-price option
   - fastest_flight: Brief description of the shortest-duration option
   - recommendation: Your top recommendation with a brief reason
   """
   ```

3. **Create the agent** with `output_schema` and `output_key`:
   ```python
   agent = LlmAgent(
       name="flight_analyzer",
       model="gemini-2.0-flash",
       instruction=instruction,
       output_schema=FlightSearchResult,
       output_key="flight_analysis",
   )
   ```

4. **Chain with a tool-calling agent** using `SequentialAgent`:
   ```python
   from google.adk.agents import SequentialAgent

   search_agent = LlmAgent(name="searcher", tools=[search_flights], ...)
   analyzer_agent = LlmAgent(name="analyzer", output_schema=FlightSearchResult, ...)

   pipeline = SequentialAgent(
       name="flight_pipeline",
       sub_agents=[search_agent, analyzer_agent],
   )
   ```

5. **Read structured output from state** in downstream agents:
   ```python
   analysis = callback_context.state.get("flight_analysis")
   if analysis:
       parsed = FlightSearchResult.model_validate_json(analysis)
   ```

6. **Handle validation failures**: ADK retries up to 3 times if the model
   produces invalid JSON. Add a fallback in `after_agent_callback` if all
   retries fail.

## Input Format

```python
{
  "agent_goal": "Extract structured flight analysis from search results",
  "user_input": "Analyze the flight options found",
  "context": {"search_results": "[...raw flight data...]"},
  "tools": [],
  "output_format": "FlightSearchResult schema",
  "constraints": ["All fields required", "recommendation max 50 words"]
}
```

## Output Format

```python
{
  "system_prompt": "Analyze the flight search results and populate the output schema...",
  "instructions": "...",
  "expected_output_format": {
    "type": "json",
    "schema": {
      "flights_found": "integer",
      "cheapest_flight": "string | null",
      "fastest_flight": "string | null",
      "recommendation": "string"
    }
  },
  "tool_usage_guidance": {}
}
```

## Error Handling

- **Schema validation failure** (model generates invalid JSON): ADK retries up
  to 3 times automatically. If all retries fail, the agent raises an exception.
  Add `after_agent_callback` to catch and handle gracefully.
- **Model refuses to output only JSON** (adds extra text): Add to instruction:
  "Your entire response MUST be a valid JSON object matching the schema.
  Do not include any text outside the JSON."
- **Missing required field in output**: Pydantic raises `ValidationError`.
  Make fields `Optional` with defaults for fields the model may not always
  populate.
- **Instruction and schema conflict** (instruction says one thing, schema allows
  another): The schema is the source of truth for structure. The instruction
  defines content. Align them.
- **Tool + output_schema conflict**: ADK does not allow both simultaneously.
  Use `SequentialAgent`: tool-calling agent → extraction agent.

## Examples

### Example 1 — Extraction agent with Pydantic schema

```python
from google.adk.agents import LlmAgent, SequentialAgent
from pydantic import BaseModel, Field
from typing import Optional, List

class FlightAnalysis(BaseModel):
    flights_found: int = Field(description="Total number of flights available")
    cheapest_price_usd: Optional[float] = Field(description="Price of cheapest flight in USD")
    fastest_duration_minutes: Optional[int] = Field(description="Duration of fastest flight in minutes")
    recommended_flight: str = Field(description="Name/code of the recommended flight")
    reasoning: str = Field(description="Brief reason for the recommendation (max 50 words)")

ANALYSIS_INSTRUCTION = """
You are a flight analysis agent. You will receive flight search results in session state.
Analyze the results and populate each field of the output schema:

- flights_found: Count the number of flights in the results
- cheapest_price_usd: Extract the lowest price as a float
- fastest_duration_minutes: Extract the shortest duration in minutes
- recommended_flight: Choose the best overall flight considering price and duration
- reasoning: Explain your recommendation in 30-50 words

Base your analysis ONLY on the session state data. Do not invent data.
"""

flight_searcher = LlmAgent(
    name="flight_searcher",
    model="gemini-2.0-flash",
    instruction="Search for flights using search_flights. Store results in state.",
    tools=[search_flights],
    output_key="raw_search_results",
)

flight_analyzer = LlmAgent(
    name="flight_analyzer",
    model="gemini-2.0-flash",
    instruction=ANALYSIS_INSTRUCTION,
    output_schema=FlightAnalysis,
    output_key="flight_analysis",
)

pipeline = SequentialAgent(
    name="flight_pipeline",
    sub_agents=[flight_searcher, flight_analyzer],
)
```

### Example 2 — Nested schema for complex extraction

```python
from pydantic import BaseModel
from typing import List

class PropertyFeatures(BaseModel):
    bedrooms: int
    bathrooms: float
    square_feet: int
    has_garage: bool
    has_pool: bool

class PropertyListing(BaseModel):
    address: str
    price_usd: int
    features: PropertyFeatures
    neighborhood_score: float  # 0.0 to 10.0
    investment_rating: str     # "strong_buy", "buy", "hold", "avoid"
    summary: str               # Max 100 words

EXTRACTION_INSTRUCTION = """
Extract property listing data from the listing_text in session state.
Populate all fields accurately. For neighborhood_score, use the score from
the neighborhood_data tool result in state. For investment_rating, assess
based on price-to-value ratio and neighborhood score.
"""

property_extractor = LlmAgent(
    name="property_extractor",
    model="gemini-2.0-pro",
    instruction=EXTRACTION_INSTRUCTION,
    output_schema=PropertyListing,
    output_key="property_listing",
)
```

### Example 3 — Reading structured output in next pipeline step

```python
from typing import Optional
from google.adk.agents.callback_context import CallbackContext
from google.genai.types import Content

def read_analysis_in_callback(callback_context: CallbackContext) -> Optional[Content]:
    """Reads structured flight analysis from state for downstream processing."""
    analysis_raw = callback_context.state.get("flight_analysis")
    if not analysis_raw:
        return None
    analysis = FlightAnalysis.model_validate_json(
        analysis_raw if isinstance(analysis_raw, str) else str(analysis_raw)
    )
    # Inject summary into state for use in next agent
    callback_context.state["booking_recommendation"] = analysis.recommended_flight
    callback_context.state["booking_reasoning"] = analysis.reasoning
    return None
```

## Edge Cases

- **Optional vs. Required fields**: Start with `Optional` fields and default
  values. Tighten to required after testing confirms the model reliably populates them.
- **Enum fields**: Use `Literal` or `Enum` types in Pydantic for constrained
  string fields (e.g., `investment_rating: Literal["strong_buy", "buy", "hold", "avoid"]`).
- **Large nested schemas** (> 10 fields): Split into separate extraction agents
  per sub-schema, each with a focused instruction.
- **Schema changes break existing sessions**: Store schema version in `app:` state.
  Validate `output_key` values against the current schema version before use.

## Integration Notes

- **`LlmAgent.output_key`**: Automatically stores structured output in
  `session.state[output_key]`. Downstream agents access via `{output_key?}`.
- **`google-adk-output-formatting`**: For text-based output formatting (user-facing),
  use that skill. For machine-parseable output, use this skill.
- **`SequentialAgent`**: Required pattern for tool-calling + structured output
  pipelines. The structured output agent runs after the tool-calling agent.
- **`google-adk-chain-of-thought-prompting`**: Add a `reasoning: str` field
  to the schema to capture chain-of-thought alongside the structured answer.
