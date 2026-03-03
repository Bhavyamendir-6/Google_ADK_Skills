---
name: prompt-optimization
description: >
  Use this skill when improving the quality and efficiency of prompts (system
  instructions) used by Google ADK LlmAgent to maximize agent reasoning quality,
  reduce token consumption, improve tool-calling accuracy, and minimize
  multi-turn reasoning loops. Covers instruction clarity, role definition,
  constraint specification, few-shot examples, output format directives,
  and iterative prompt refinement with ADK eval.
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

This skill optimizes the prompts (system instructions) passed to Google ADK
`LlmAgent` to improve agent reasoning quality, reduce unnecessary multi-turn
loops, improve tool-call accuracy, and minimize token waste. A poorly structured
instruction can cause: incorrect tool selection, excessive reasoning turns,
verbose outputs, ignored constraints, and hallucinated information. This skill
applies structured prompt engineering principles (role clarity, constraint blocks,
output format specification, few-shot examples) specifically within the ADK
`LlmAgent.instruction` and `InstructionProvider` patterns.

## When to Use

- When the agent makes incorrect tool calls or selects the wrong tool.
- When the agent requires multiple reasoning turns for tasks that should complete
  in one turn.
- When the agent produces verbose outputs that consume excessive output tokens.
- When the agent ignores user preferences or constraints injected via state.
- When the agent hallucinates information not present in retrieved context.
- When quality eval scores are below baseline and no other optimization applies.

## When NOT to Use

- Do not apply arbitrary prompt changes without eval validation — prompt changes
  can silently degrade quality.
- Do not add more instruction text as the first solution — first try making
  existing instructions clearer.
- Do not embed dynamic data in static instructions — use `{key?}` injection.

## Google ADK Context

- **`LlmAgent.instruction`**: A string (static) or callable `InstructionProvider`
  returning a string. Fed as the system instruction.
- **`InstructionProvider`**: `def instruction(ctx: ReadonlyContext) -> str`—enables
  dynamic instruction construction from session state.
- **`{key?}` injection**: Dynamic state values injected into instruction template.
  Keep injected values compact.
- **Tool docstrings as prompt**: Each tool's docstring is the tool description in
  the prompt. Concise, accurate docstrings improve tool selection.
- **`output_schema`**: Structured output constrains the model to produce valid
  JSON in a specific format — more reliable than free-text parsing.
- **ADK multi-turn reasoning**: Each incorrect or ambiguous tool call adds a
  reasoning turn. Clear, specific tool guidance reduces turns.
- **Eval-driven optimization**: Use `AgentEvaluator` with `response_match_score`
  and `tool_trajectory_avg_score` to measure prompt quality objectively.

## Capabilities

- Structures instructions with clear role, task, constraints, and format sections.
- Uses `output_schema` for reliable structured output instead of free-text parsing.
- Writes concise, accurate tool docstrings to improve tool selection.
- Adds a minimal set of few-shot examples for complex tasks.
- Applies `InstructionProvider` for step-adaptive instructions.
- Measures prompt quality with ADK eval metrics.
- Reduces multi-turn reasoning loops via explicit tool selection guidance.

## Step-by-Step Instructions

1. **Audit the current instruction** for: role clarity, task ambiguity, constraint
   completeness, output format specification.

2. **Structure the instruction** using the RCTF pattern:
   - **R**ole: "You are FlightBot, an airline booking assistant."
   - **C**onstraints: "Only book flights via the search_flights and book_flight tools. Never invent flight data."
   - **T**ask: "Help the user search and book flights."
   - **F**ormat: "Respond concisely in 1–3 sentences unless the user asks for detail."

3. **Write precise tool docstrings** (the tool summary the model sees):
   ```python
   def search_flights(origin: str, destination: str, date: str, tool_context: ToolContext) -> dict:
       """Searches for available flights between two airports on a given date.
       Args:
           origin (str): IATA origin airport code (e.g., 'JFK').
           destination (str): IATA destination airport code (e.g., 'LHR').
           date (str): Travel date in YYYY-MM-DD format.
       Returns:
           dict: Contains 'flights' (list), each with 'airline', 'flight_number', 'price', 'departs', 'arrives'.
       """
   ```

4. **Use `output_schema`** for tasks requiring structured output:
   ```python
   class BookingConfirmation(BaseModel):
       flight_number: str
       departure: str
       arrival: str
       price: float
       confirmed: bool

   agent = LlmAgent(output_schema=BookingConfirmation, ...)
   ```

5. **Add few-shot examples** for complex tool-call sequences (max 2–3 examples):
   ```python
   instruction = """...
   EXAMPLE:
   User: Book the cheapest flight from JFK to LHR next Tuesday.
   Action: call search_flights(origin='JFK', destination='LHR', date='2025-03-11')
   Action: call book_flight(flight_number='BA178', passenger_name='{user_name?}')
   """
   ```

6. **Run ADK eval** to measure improvement:
   ```
   adk eval my_agent/ evals/booking_eval.json
   ```

## Input Format

```python
{
  "operation": "instruction_optimization",
  "execution_time_ms": 8200,
  "token_usage": 4200,
  "memory_usage_mb": 32,
  "cost_estimate": 0.0031,
  "retrieval_latency_ms": 0,
  "cache_available": False,
  "parallelizable": False,
  "context_size_tokens": 4200
}
```

## Output Format

```python
{
  "optimization_applied": True,
  "optimization_type": "instruction_restructuring + docstring_optimization + output_schema",
  "performance_improvement": {
    "latency_reduction_ms": 2100,
    "token_reduction": 1400,
    "memory_reduction_mb": 0,
    "cost_reduction": 0.0007,
    "throughput_improvement": 1.6
  }
}
```

## Error Handling

- **Instruction too long after structuring** (> 1000 tokens): Remove few-shot
  examples first. Move constraints into tool docstrings. Use `InstructionProvider`
  to inject only what is needed per turn.
- **`output_schema` causes validation errors** (model produces slightly wrong JSON):
  Add a Pydantic `model_validator` with permissive field parsing. Or use Flash
  (`gemini-2.0-flash`) which has stronger structured output enforcement.
- **Tool docstring change breaks tool selection**: The model learned from the old
  docstring. Update incrementally. Run eval after each change.
- **Few-shot examples inflate tokens significantly**: Limit to 2 examples. Use
  minimal examples (short user query + expected action). Remove if token budget
  is tight.

## Examples

### Example 1 — Well-structured RCTF instruction

```python
from google.adk.agents import LlmAgent
from google.adk.agents.readonly_context import ReadonlyContext
from google.genai.types import GenerateContentConfig

def optimized_instruction(ctx: ReadonlyContext) -> str:
    """Builds a structured, minimal instruction dynamically."""
    step = ctx.state.get("booking_step", "search")
    user_name = ctx.state.get("user:name", "the user")

    role = f"You are FlightBot, AcmeAir's booking assistant for {user_name}."
    constraints = (
        "CONSTRAINTS:\n"
        "- Only provide information from tool results. Never invent flight data.\n"
        "- Use search_flights before recommending any flight.\n"
        "- Confirm details before calling book_flight.\n"
        "- Respond in ≤3 sentences unless the user asks for detail."
    )
    task = f"CURRENT STEP: {step}. Guide the user to complete their booking."

    active_context = ""
    if step == "confirm":
        flight = ctx.state.get("selected_flight", "")
        price = ctx.state.get("selected_price", "")
        active_context = f"\nSELECTED: {flight} at ${price}. Confirm with the user."

    return f"{role}\n\n{constraints}\n\n{task}{active_context}"

optimized_agent = LlmAgent(
    name="flight_booking_agent",
    model="gemini-2.0-flash",
    instruction=optimized_instruction,
    tools=[search_flights, book_flight, process_payment, get_booking_status],
    generate_content_config=GenerateContentConfig(temperature=0.2, max_output_tokens=400),
)
```

## Edge Cases

- **Instruction version drift** (prompt changed in DEV but not PROD): Track
  instruction version in `app:instruction_version` state. Log version mismatches.
- **Few-shot examples contain outdated tool call syntax**: Update examples whenever
  tool signatures change. Examples with wrong syntax mislead the model.
- **User speaks a different language** from instruction: Add localization instruction:
  "Respond in the same language as the user's message."

## Integration Notes

- **`google-adk-token-usage-optimization`**: Concise instructions reduce per-turn
  token cost.
- **`google-adk-latency-optimization`**: Clearer instructions reduce multi-turn
  reasoning loops, cutting latency.
- **`google-adk-model-selection-optimization`**: Good prompt engineering can
  compensate for a model downgrade.
- **`google-adk-automated-evaluation`**: All prompt changes must be validated with
  `AgentEvaluator` before production deployment.
