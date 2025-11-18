# Streaming System - Deep Dive

Real-time streaming architecture for agent responses with code references.

## Overview

Letta implements multiple streaming patterns: Server-Sent Events (SSE), token-by-token streaming, and resumable streaming for long-running operations.

**Primary Files:**
- [`letta/server/rest_api/interface.py`](../letta-repo/letta/server/rest_api/interface.py) - Streaming interfaces
- [`letta/interfaces/openai_streaming_interface.py`](../letta-repo/letta/interfaces/openai_streaming_interface.py) - OpenAI streaming
- [`letta/interfaces/anthropic_streaming_interface.py`](../letta-repo/letta/interfaces/anthropic_streaming_interface.py) - Anthropic streaming

---

## Streaming Types

### 1. Step Streaming (No Tokens)

**Fast streaming**: Agent steps without token-by-token streaming.

```python
# Location: letta/agents/letta_agent.py:208-549
async def step_stream_no_tokens(
    self,
    input_messages: List[MessageCreate],
    max_steps: int = DEFAULT_MAX_STEPS,
) -> AsyncGenerator:
    """
    Stream agent steps without token streaming.

    Yields:
    - LettaMessage objects as they're created
    - Stop reason
    - Usage statistics
    """

    for i in range(max_steps):
        # ... agent step execution ...

        # Yield messages as they're created
        for message in persisted_messages:
            yield f"data: {message.model_dump_json()}\n\n"

        if not should_continue:
            break

    # Yield final stop reason
    yield f"data: {stop_reason.model_dump_json()}\n\n"

    # Yield usage stats
    yield f"data: {usage.model_dump_json()}\n\n"

    # Done
    yield "data: [DONE]\n\n"
```

### 2. Token-by-Token Streaming

**Full streaming**: Stream LLM tokens as they're generated.

```python
# Location: letta/agents/letta_agent.py:892-1632
async def step_stream(
    self,
    input_messages: List[MessageCreate],
    stream_tokens: bool = True,
) -> AsyncGenerator:
    """
    Stream agent execution with token-by-token streaming.

    Yields:
    - Token chunks (individual tokens as generated)
    - Tool calls
    - Tool results
    - Final messages
    """

    # Call LLM with streaming enabled
    stream = await llm_client.create_chat_completion_stream_async(
        messages=request_messages,
        tools=tools,
        stream=True,
    )

    # Stream tokens as they arrive
    async for chunk in stream:
        if chunk.choices[0].delta.content:
            yield f"data: {chunk.model_dump_json()}\n\n"
```

### 3. Resumable Streaming

**For long-running operations**: Resume streaming if connection drops.

```python
# Create agent message with background mode
stream = client.agents.messages.create_stream(
    agent_id=agent.id,
    messages=[...],
    background=True,                         # Enable background mode
)

run_id = None
last_seq_id = None

for chunk in stream:
    if hasattr(chunk, "run_id"):
        run_id = chunk.run_id                # Save for reconnection
        last_seq_id = chunk.seq_id           # Resumption point
    print(chunk)

# If disconnected, resume from last seq_id
for chunk in client.runs.stream(run_id, starting_after=last_seq_id):
    print(chunk)
```

---

## Streaming Interfaces

### OpenAI Streaming Interface

**Location:** [`letta/interfaces/openai_streaming_interface.py`](../letta-repo/letta/interfaces/openai_streaming_interface.py)

```python
class OpenAIStreamingInterface:
    """
    Handles streaming for OpenAI-compatible responses.

    Converts agent execution into OpenAI-format SSE events.
    """

    async def stream_response(
        self,
        agent: LettaAgent,
        messages: List[Message],
    ) -> AsyncGenerator:
        """
        Stream agent response in OpenAI format.

        Format:
        data: {"id":"chatcmpl-123","object":"chat.completion.chunk",...}
        """

        async for chunk in agent.step_stream(messages):
            # Convert to OpenAI format
            openai_chunk = self._convert_to_openai_format(chunk)
            yield f"data: {openai_chunk}\n\n"

        yield "data: [DONE]\n\n"
```

### Anthropic Streaming Interface

**Location:** [`letta/interfaces/anthropic_streaming_interface.py`](../letta-repo/letta/interfaces/anthropic_streaming_interface.py)

```python
class AnthropicStreamingInterface:
    """
    Handles streaming for Anthropic Claude responses.

    Anthropic uses different event types:
    - message_start
    - content_block_start
    - content_block_delta
    - content_block_stop
    - message_stop
    """

    async def stream_response(
        self,
        agent: LettaAgent,
        messages: List[Message],
    ) -> AsyncGenerator:
        """Stream in Anthropic format."""

        # Emit message_start event
        yield self._create_message_start_event()

        async for chunk in agent.step_stream(messages):
            # Convert to Anthropic event format
            event = self._convert_to_anthropic_event(chunk)
            yield f"event: {event.type}\ndata: {event.data}\n\n"

        # Emit message_stop event
        yield self._create_message_stop_event()
```

---

## Server-Sent Events (SSE)

### FastAPI SSE Implementation

**Location:** [`letta/server/rest_api/routers/v1/messages.py`](../letta-repo/letta/server/rest_api/routers/v1/messages.py)

```python
from fastapi.responses import StreamingResponse

@router.post("/{agent_id}/messages")
async def create_agent_message(
    agent_id: str,
    request: MessageCreate,
    stream: bool = False,
):
    """Send message with optional streaming."""

    if stream:
        return StreamingResponse(
            _stream_generator(agent_id, request),
            media_type="text/event-stream",   # SSE content type
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",    # Disable nginx buffering
            }
        )
    else:
        # Non-streaming response
        response = await agent.step(messages=[request])
        return response

async def _stream_generator(agent_id: str, request: MessageCreate):
    """Async generator for SSE."""

    agent = await agent_manager.get_agent_by_id_async(agent_id, ...)

    async for chunk in agent.step_stream_no_tokens(
        input_messages=[request]
    ):
        yield chunk  # Already formatted as "data: ...\n\n"
```

---

## Event Format

### SSE Message Structure

```
event: message_type
data: {"key": "value"}

```

**Standard Events:**

```
# Token chunk
data: {"type":"chat_completion_chunk","delta":{"content":"Hello"}}

# Function call
data: {"message_type":"function_call","name":"memory","arguments":{...}}

# Function return
data: {"message_type":"function_return","status":"success","output":"Done"}

# Stop reason
data: {"stop_reason":"end_turn"}

# Usage stats
data: {"completion_tokens":42,"prompt_tokens":1500,"total_tokens":1542}

# Done marker
data: [DONE]

```

---

## Client-Side Consumption

### JavaScript Example

```javascript
const eventSource = new EventSource(
    'http://localhost:8283/v1/agents/agent-123/messages?stream=true'
);

eventSource.onmessage = (event) => {
    if (event.data === '[DONE]') {
        eventSource.close();
        return;
    }

    const chunk = JSON.parse(event.data);
    console.log(chunk);

    if (chunk.type === 'chat_completion_chunk') {
        // Append token to UI
        appendToken(chunk.delta.content);
    }
};

eventSource.onerror = (error) => {
    console.error('SSE error:', error);
    eventSource.close();
};
```

### Python Client Example

```python
from letta_client import Letta

client = Letta(token="...")

# Streaming mode
async for chunk in client.agents.messages.create_stream(
    agent_id="agent-123",
    messages=[{"role": "user", "content": "Hello"}],
    stream_tokens=True,
):
    print(chunk)
```

---

## Performance Considerations

### Buffering

Disable buffering at all layers for real-time streaming:

1. **Nginx**: `X-Accel-Buffering: no`
2. **FastAPI**: `StreamingResponse` with no buffer
3. **Python**: Yield frequently, don't accumulate

### Backpressure

Handle slow clients with backpressure:

```python
async def stream_with_backpressure(agent_id: str, request: MessageCreate):
    """Stream with client backpressure handling."""

    buffer = asyncio.Queue(maxsize=10)  # Limit buffer size

    async def producer():
        async for chunk in agent.step_stream(messages):
            await buffer.put(chunk)
        await buffer.put(None)  # Signal done

    async def consumer():
        while True:
            chunk = await buffer.get()
            if chunk is None:
                break
            yield chunk

    # Run producer in background
    asyncio.create_task(producer())

    # Stream from consumer
    async for chunk in consumer():
        yield chunk
```

---

## Error Handling

### Graceful Disconnection

```python
try:
    async for chunk in agent.step_stream(messages):
        yield chunk
except asyncio.CancelledError:
    # Client disconnected
    logger.info("Client disconnected, cleaning up...")
    await agent.cancel_run()
    raise
except Exception as e:
    # Error during streaming
    error_event = {
        "type": "error",
        "error": str(e)
    }
    yield f"data: {json.dumps(error_event)}\n\n"
```

---

## See Also

- [Agent Execution](./agent-execution.md) - How streaming integrates with execution
- [API Layer](./api-layer.md) - HTTP streaming endpoints
- [LLM Integration](./llm-integration.md) - Provider-specific streaming
