---
name: structured-tool-inputs
description: >
  Use this skill when you need to define, validate, and coerce structured
  inputs for agent tool calls. Ensures LLM-generated arguments conform to
  expected schemas before execution using Pydantic or JSON Schema validation.
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

This skill defines how to structure tool inputs using typed schemas (Pydantic
models or dataclasses) and how to validate LLM-generated arguments before
passing them to tool functions. It prevents runtime failures caused by missing,
malformed, or type-mismatched arguments from model-generated tool calls.

## When to Use

- When a tool accepts multiple parameters and argument correctness is critical.
- When the LLM frequently hallucinates or misformats tool arguments.
- When building production tools that must reject invalid inputs gracefully.
- When tool inputs need coercion (e.g., string `"42"` → int `42`).

## When NOT to Use

- Do not use for simple single-parameter tools where inline type annotations
  suffice.
- Do not use to validate tool *outputs* (see `tool-result-parsing`).
- Do not replace tool definitions with schema-only objects—the tool function
  must still exist.

## Capabilities

- Defines Pydantic `BaseModel` input schemas for tool parameters.
- Validates and coerces LLM-generated argument dicts against the schema.
- Provides structured error messages for invalid inputs.
- Integrates with Google ADK's schema extraction pipeline.
- Supports nested objects, enums, and constrained types.

## Step-by-Step Instructions

1. **Identify all parameters** for the tool function including types and
   constraints (min/max, allowed values, regex patterns).

2. **Create a Pydantic `BaseModel`** named `<ToolName>Input` with each
   parameter as a typed field:
   ```python
   from pydantic import BaseModel, Field
   class GetStockPriceInput(BaseModel):
       ticker: str = Field(..., description="Stock ticker symbol, e.g. 'AAPL'")
       currency: str = Field("USD", description="Currency code, e.g. 'USD'")
   ```

3. **Use `Field(...)` for required fields** and `Field(default)` for optional.
   Always include a `description` for every field.

4. **Add validators** for constrained fields:
   ```python
   from pydantic import field_validator
   @field_validator("ticker")
   def ticker_must_be_uppercase(cls, v: str) -> str:
       return v.upper()
   ```

5. **Update the tool function signature** to accept the Pydantic model or
   individual validated fields. Prefer individual fields for framework
   compatibility.

6. **At call time**, validate the raw args dict before passing to the function:
   ```python
   validated = GetStockPriceInput(**raw_args)
   result = get_stock_price(**validated.model_dump())
   ```

7. **Handle `ValidationError`** by returning a structured error message as the
   tool result (do not raise).

## Input Format

```
tool_name: <str>
raw_args: dict  # LLM-generated argument dict, may be invalid
schema: <PydanticModel class>
```

## Output Format

On success:
```python
validated_args: dict  # Fully validated and coerced arguments
```

On failure:
```python
{
  "error": "Validation failed",
  "details": [
    {"field": "ticker", "message": "field required"},
    {"field": "currency", "message": "value is not a valid currency code"}
  ]
}
```

## Error Handling

- Catch `pydantic.ValidationError` and convert to the dict format above.
- Return the error dict as the tool result (inserted into model context).
- Do NOT raise `ValidationError` up the call stack—it must be handled here.
- Log validation failures with the raw args for debugging.

## Examples

### Example 1 — Simple Pydantic input model

```python
from pydantic import BaseModel, Field, field_validator
from typing import Literal

class GetWeatherInput(BaseModel):
    city: str = Field(..., description="City name, e.g. 'London'")
    unit: Literal["celsius", "fahrenheit"] = Field(
        "celsius", description="Temperature unit"
    )

    @field_validator("city")
    def city_must_not_be_empty(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("City name must not be empty")
        return v.strip().title()
```

### Example 2 — Validation wrapper function

```python
from pydantic import ValidationError
import json

def validated_tool_call(schema_cls, raw_args: dict, fn) -> dict:
    """Validates raw_args against schema_cls, then calls fn."""
    try:
        validated = schema_cls(**raw_args)
    except ValidationError as e:
        return {
            "error": "Validation failed",
            "details": [
                {"field": err["loc"][0], "message": err["msg"]}
                for err in e.errors()
            ],
        }
    return fn(**validated.model_dump())
```

### Example 3 — Integration in agent loop

```python
result = validated_tool_call(
    schema_cls=GetWeatherInput,
    raw_args={"city": "london", "unit": "fahrenheit"},
    fn=get_weather,
)
```

## Edge Cases

- **LLM sends string for integer field**: Pydantic coerces `"42"` → `42`
  automatically in lax mode. Use `model_config = ConfigDict(strict=False)`.
- **LLM sends extra fields**: Set `model_config = ConfigDict(extra="ignore")`
  to silently discard unknown keys.
- **Nested object arguments**: Define nested `BaseModel` classes and use them
  as field types. Pydantic handles nested validation automatically.
- **Enum validation**: Use `Literal["a", "b"]` or `Enum` classes. The
  validation error message will include the allowed values.

## Integration Notes

- **Google ADK**: ADK extracts JSON Schema from type annotations. Using Pydantic
  models as parameter types improves schema richness and validation.
- **OpenAI Agents SDK**: Pass a Pydantic model to `function_tool(args_type=...)`
  for automatic schema extraction and validation.
- **MCP**: Define `inputSchema` using Pydantic's `.model_json_schema()` output
  when registering MCP tools.
- **function-calling skill**: Apply this skill in step 4 of the function-calling
  loop, before dispatching to the function.
