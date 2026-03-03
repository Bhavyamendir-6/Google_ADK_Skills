---
name: streaming-responses
description: >
  Use this skill when configuring a Google ADK LlmAgent to stream model
  responses token-by-token for lower perceived latency. Covers consuming ADK
  Runner event streams for incremental text delivery, handling streaming-specific
  event types, implementing graceful stream interruption handling, and combining
  streaming with tool calls and structured output.
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

This skill implements streaming response delivery for Google ADK `LlmAgent`.
ADK's `Runner.run()` is a generator that yields `Event` objects as the agent
executes — including partial text events, tool call events, and final response
events. By consuming partial text events incrementally, applications can display
agent responses token-by-token, dramatically reducing the perceived latency from
the first token to user display. This skill covers the event stream consumption
pattern, streaming-specific event handling, interruption recovery, and integration
with streaming-capable frontends.

## When to Use

- When building user-facing chat interfaces that need real-time streaming output.
- When P95 latency to first visible token is a key UX metric.
- When the agent's final responses are long (> 200 words) and users should not
  wait for the full response before seeing any content.
- When integrating ADK agents with WebSocket or Server-Sent Events (SSE) APIs
  for real-time frontend delivery.

## When NOT to Use

- Do not implement streaming for machine-to-machine communication where the
  full response must be received before processing — consume the full
  `is_final_response()` event instead.
- Do not apply streaming to structured output agents where the JSON must be
  valid before parsing — wait for the final response event.
- Do not use streaming in batch evaluation runs — it adds complexity without
  latency benefit in offline evaluation.

## Google ADK Context

- **`Runner.run()`**: Returns a Python generator of `Event` objects. Yields
  events as they are produced by the model and tool execution.
- **ADK Event stream types**:
  - Partial text events: `event.content.parts` contains text chunks; `event.is_final_response() == False`
  - Tool call events: `event.content.parts` contains function calls.
  - Tool response events: Contains `function_response` parts.
  - Final response event: `event.is_final_response() == True` — marks the end.
- **Streaming consumption pattern**: Iterate the generator, check for text parts
  on each event, and deliver partial text to the UI as it arrives.
- **`event.partial`**: Indicates whether the event is a partial (streaming) chunk.
- **Tool calls during streaming**: The event stream pauses text streaming during
  tool execution. Text streaming resumes after the tool returns.
- **Streaming + `output_schema`**: Structured output requires the full JSON
  before validation — always wait for `is_final_response()` for schema parsing.
- **Stop sequences in streaming**: The stream is cut at the stop sequence token.
  Partial events stop being yielded immediately.

## Capabilities

- Consumes ADK `Runner.run()` event stream for incremental text delivery.
- Identifies and extracts partial text chunks from streaming events.
- Handles tool call events transparently during streaming.
- Implements graceful stream interruption and reconnection.
- Delivers streamed tokens to WebSocket or SSE channels.
- Combines streaming with stop sequences and token limits.

## Step-by-Step Instructions

1. **Configure the agent** as usual (no special `generate_content_config`
   needed for streaming — `Runner.run()` is always a generator):
   ```python
   agent = LlmAgent(name="chat_agent", model="gemini-2.0-flash", ...)
   runner = Runner(agent=agent, app_name="my_app", session_service=session_service)
   ```

2. **Consume the event stream** in a streaming loop:
   ```python
   for event in runner.run(user_id=uid, session_id=sid, new_message=msg):
       if event.content and event.content.parts:
           for part in event.content.parts:
               if hasattr(part, "text") and part.text:
                   yield part.text  # Deliver partial text to user
       if event.is_final_response():
           break
   ```

3. **Handle tool call events** (no text emitted during tool execution — pause UI):
   ```python
   for event in runner.run(...):
       if event.content and event.content.parts:
           for part in event.content.parts:
               if hasattr(part, "function_call"):
                   yield f"\n[Calling tool: {part.function_call.name}...]\n"
               elif hasattr(part, "text") and part.text:
                   yield part.text
   ```

4. **Implement graceful interruption** using try/finally:
   ```python
   try:
       for event in runner.run(...):
           for part in (event.content.parts or []):
               if hasattr(part, "text") and part.text:
                   yield part.text
           if event.is_final_response():
               break
   except GeneratorExit:
       pass  # Client disconnected — clean shutdown
   except Exception as exc:
       logger.error(f"Streaming error: {exc}")
       yield "\n[Stream interrupted. Please retry.]"
   ```

5. **For web delivery**, wrap in an SSE or WebSocket sender:
   ```python
   # FastAPI SSE example
   from fastapi.responses import StreamingResponse

   async def stream_agent(query: str):
       async def generate():
           for token in run_streaming(runner, uid, sid, query):
               yield f"data: {token}\n\n"
       return StreamingResponse(generate(), media_type="text/event-stream")
   ```

## Input Format

```python
{
  "model": "gemini-2.0-flash",
  "temperature": 0.4,
  "top_k": 40,
  "top_p": 0.95,
  "max_tokens": 1024,
  "stop_sequences": [],
  "stream": True,
  "response_schema": {},
  "deterministic": False
}
```

## Output Format

```python
{
  "model_used": "gemini-2.0-flash",
  "configuration_applied": {"stream": True},
  "output_valid": True,
  "structured_output": {},
  "streaming_metadata": {
    "first_token_latency_ms": 210,
    "total_tokens": 380,
    "stream_complete": True
  }
}
```

## Error Handling

- **Stream interrupted by network disconnection**: Implement retry with backoff.
  On reconnection, resume from the last complete sentence (not mid-word).
- **Tool call causes long streaming gap**: Inform the user that a tool is running.
  Yield a `"\n[Processing...]\n"` placeholder during tool execution events.
- **Streaming with `output_schema`**: The stream delivers partial JSON which cannot
  be parsed until complete. Only parse the structured output after `is_final_response()`.
- **`GeneratorExit` from client disconnect**: Catch in the streaming loop and
  log the abandoned session ID. Do not raise the exception further.
- **Empty streaming chunks**: Some events yield an empty text part. Skip with
  `if not part.text: continue`.

## Examples

### Example 1 — Complete streaming runner with FastAPI SSE

```python
import asyncio
import logging
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.agents import LlmAgent
from google.genai.types import Content, Part

logger = logging.getLogger("streaming")

def stream_agent_response(runner: Runner, user_id: str, session_id: str, query: str):
    """Generator that yields text tokens from the ADK agent as they are produced."""
    msg = Content(parts=[Part(text=query)])
    try:
        for event in runner.run(user_id=user_id, session_id=session_id, new_message=msg):
            if not event.content or not event.content.parts:
                continue
            for part in event.content.parts:
                if hasattr(part, "function_call") and part.function_call:
                    # Inform user that a tool is being called
                    yield f"\n[Using tool: {part.function_call.name}]\n"
                elif hasattr(part, "text") and part.text:
                    yield part.text
            if event.is_final_response():
                break
    except GeneratorExit:
        logger.info(f"Client disconnected. Session: {session_id}")
    except Exception as exc:
        logger.error(f"Streaming error. Session: {session_id}. Error: {exc}")
        yield "\n[An error occurred. Please retry.]"
```

### Example 2 — FastAPI SSE endpoint for streaming agent

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.get("/chat/stream")
async def chat_stream(query: str, session_id: str, user_id: str):
    """Server-Sent Events endpoint for streaming ADK agent responses."""
    async def event_generator():
        for token in stream_agent_response(runner, user_id, session_id, query):
            # SSE requires "data: " prefix and double newline
            yield f"data: {token}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(event_generator(), media_type="text/event-stream",
                              headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"})
```

### Example 3 — Measuring time-to-first-token latency

```python
import time
from google.adk.runners import Runner
from google.genai.types import Content, Part

def measure_ttft(runner: Runner, user_id: str, session_id: str, query: str) -> float:
    """Measures time-to-first-token latency for the streaming response."""
    msg = Content(parts=[Part(text=query)])
    start = time.perf_counter()
    ttft = None

    for event in runner.run(user_id=user_id, session_id=session_id, new_message=msg):
        if ttft is None and event.content and event.content.parts:
            for part in event.content.parts:
                if hasattr(part, "text") and part.text:
                    ttft = round((time.perf_counter() - start) * 1000, 2)
                    break
        if event.is_final_response():
            break

    return ttft or 0.0
```

## Edge Cases

- **Client-side buffering defeats streaming**: If the HTTP client buffers the
  entire response, streaming yields no perceived benefit. Ensure the client
  is in streaming mode and the server sets `X-Accel-Buffering: no`.
- **Very fast responses** (< 50ms total): Streaming overhead may exceed the
  response time. Use non-streaming for near-instant responses.
- **Multiple tool calls in one turn**: The stream pauses at each tool call and
  resumes after the response. Implement a "tool call indicator" in the UI.
- **Stop sequence cuts streaming mid-word**: Stop sequences are byte-level.
  If a stop sequence coincides with a multi-byte Unicode character boundary,
  the partial character may cause encoding issues. Use ASCII-safe stop sequences.

## Integration Notes

- **`google-adk-token-limits`**: `max_output_tokens` applies to streaming
  identically. The stream is cut at the token limit.
- **`google-adk-stop-sequences`**: Stop sequences cut the stream immediately.
  Combine with stop sequences for clean stream termination.
- **`google-adk-structured-json-output`**: Never parse structured output from
  partial stream events — always wait for `is_final_response()`.
- **`google-adk-latency-optimization`**: Streaming reduces perceived latency
  (time-to-first-token) even when total generation time is unchanged.
