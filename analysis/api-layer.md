# API Layer - Deep Dive

REST API architecture with code references and endpoint details.

## Overview

Letta provides a production-ready REST API built on FastAPI with 70+ endpoints, streaming support, and OpenAPI documentation.

**Primary Files:**
- [`letta/server/rest_api/app.py`](../letta-repo/letta/server/rest_api/app.py) - FastAPI application
- [`letta/server/rest_api/routers/v1/`](../letta-repo/letta/server/rest_api/routers/v1/) - Route handlers
- [`letta/server/server.py`](../letta-repo/letta/server/server.py) - Server orchestrator

---

## API Structure

### Endpoint Groups

**Location:** [`letta/server/rest_api/routers/v1/`](../letta-repo/letta/server/rest_api/routers/v1/)

```
routers/v1/
├── agents.py          # Agent CRUD, configuration
├── messages.py        # Send messages, streaming
├── blocks.py          # Memory block management
├── tools.py           # Tool management
├── runs.py            # Execution runs, metrics
├── sources.py         # Data sources, folders
├── files.py           # File uploads, metadata
├── jobs.py            # Background job status
├── mcp_servers.py     # MCP server management
├── llms.py            # LLM provider config
└── admin.py           # Admin endpoints
```

---

## Key Endpoints

### Agents

```http
POST   /v1/agents                    # Create agent
GET    /v1/agents                    # List agents
GET    /v1/agents/{agent_id}         # Get agent
PATCH  /v1/agents/{agent_id}         # Update agent
DELETE /v1/agents/{agent_id}         # Delete agent
POST   /v1/agents/{agent_id}/export  # Export agent (.af file)
POST   /v1/agents/import             # Import agent
```

### Messages (Conversation)

```http
POST   /v1/agents/{agent_id}/messages          # Send message (streaming)
GET    /v1/agents/{agent_id}/messages          # List messages
PATCH  /v1/agents/{agent_id}/messages/{id}     # Update message
DELETE /v1/agents/{agent_id}/messages/{id}     # Delete message
POST   /v1/agents/{agent_id}/messages/cancel   # Cancel run
```

### Memory

```http
GET    /v1/agents/{agent_id}/core-memory                    # Get all blocks
GET    /v1/agents/{agent_id}/core-memory/blocks             # List blocks
GET    /v1/agents/{agent_id}/core-memory/blocks/{label}     # Get block
PATCH  /v1/agents/{agent_id}/core-memory/blocks/{label}     # Update block
PATCH  /v1/agents/{agent_id}/core-memory/blocks/attach/{id} # Attach block
PATCH  /v1/agents/{agent_id}/core-memory/blocks/detach/{id} # Detach block
```

---

## Streaming Implementation

### Server-Sent Events (SSE)

**Location:** [`letta/server/rest_api/routers/v1/messages.py`](../letta-repo/letta/server/rest_api/routers/v1/messages.py)

```python
@router.post(
    "/{agent_id}/messages",
    response_model=LettaResponse,
)
async def create_agent_message(
    agent_id: str,
    request: MessageCreate,
    stream: bool = False,              # Enable streaming
    stream_tokens: bool = False,       # Token-by-token streaming
):
    """Send message to agent."""

    if stream:
        # Return streaming response
        return StreamingResponse(
            _stream_agent_response(agent_id, request, stream_tokens),
            media_type="text/event-stream"
        )
    else:
        # Return complete response
        response = await agent.step(input_messages=[request])
        return response

async def _stream_agent_response(
    agent_id: str,
    request: MessageCreate,
    stream_tokens: bool
):
    """
    Async generator for streaming responses.

    Yields SSE-formatted events:
    - data: {message}\n\n
    - data: {stop_reason}\n\n
    - data: {usage}\n\n
    """
    async for chunk in agent.step_stream(
        input_messages=[request],
        stream_tokens=stream_tokens
    ):
        yield f"data: {chunk.model_dump_json()}\n\n"
```

---

## Authentication & Authorization

### Actor Pattern

Every request is scoped to a "user" (actor) for multi-tenancy.

**Location:** [`letta/server/rest_api/dependencies.py`](../letta-repo/letta/server/rest_api/dependencies.py)

```python
async def get_current_actor(
    token: str = Depends(oauth2_scheme)
) -> User:
    """
    Extract user (actor) from API token.

    All database operations are scoped to this actor.
    """
    user = await auth_manager.verify_token(token)
    return user
```

### Password Protection

**Configuration:** `settings.server_password`

Optional server-level password for simple auth.

---

## Response Models

### Standard Response

```python
class LettaResponse(BaseModel):
    messages: List[LettaMessage]           # All messages in response
    usage: LettaUsageStatistics            # Token usage
    stop_reason: LettaStopReason           # Why execution stopped
    run_id: str                            # Execution run ID
```

### Streaming Events

```python
# Event 1: Token chunk
data: {"type":"chat_completion_chunk","delta":{"content":"Hello"}}

# Event 2: Tool call
data: {"message_type":"function_call","function_call":{"name":"memory",...}}

# Event 3: Tool result
data: {"message_type":"function_return","function_return":"Success"}

# Event 4: Stop reason
data: {"stop_reason":"end_turn"}

# Event 5: Usage stats
data: {"completion_tokens":42,"prompt_tokens":1500}

# Event 6: Done
data: [DONE]
```

---

## Middleware

**Location:** [`letta/server/rest_api/middleware/`](../letta-repo/letta/server/rest_api/middleware/)

1. **CORS** - Cross-origin resource sharing
2. **Logging** - Request/response logging
3. **Profiling** - Performance profiling
4. **Error Handling** - Standardized error responses

---

## OpenAPI Documentation

Letta auto-generates OpenAPI docs via FastAPI.

**Access:**
- Swagger UI: `http://localhost:8283/docs`
- ReDoc: `http://localhost:8283/redoc`
- OpenAPI JSON: `http://localhost:8283/openapi.json`

---

## See Also

- [Agent Execution](./agent-execution.md) - How API calls trigger agent execution
- [Streaming System](./streaming-system.md) - Detailed streaming architecture
