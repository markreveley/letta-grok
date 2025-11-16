# Letta Quick Reference Guide

A fast-lookup reference for developers working with the Letta codebase.

## Table of Contents
- [File Locations](#file-locations)
- [Key Classes](#key-classes)
- [Common Workflows](#common-workflows)
- [Configuration](#configuration)
- [Database Models](#database-models)
- [API Endpoints Cheatsheet](#api-endpoints-cheatsheet)
- [Tool Development](#tool-development)
- [Debugging Tips](#debugging-tips)

---

## File Locations

### Core Agent Implementation
```
letta/agents/
├── base.py                    # BaseAgent abstract class
├── letta_agent.py            # Main LettaAgent implementation
├── letta_agent_v2.py         # Enhanced version
└── letta_agent_v3.py         # Latest iteration
```

### Service Managers
```
letta/services/
├── agent_manager.py          # Agent lifecycle management (3,299 LOC)
├── message_manager.py        # Message persistence (1,219 LOC)
├── tool_manager.py           # Tool CRUD operations (1,043 LOC)
├── block_manager.py          # Memory block management (809 LOC)
├── file_manager.py           # File uploads and metadata
├── run_manager.py            # Execution run tracking
└── ...                       # 30+ total managers
```

### REST API Routes
```
letta/server/rest_api/routers/v1/
├── agents.py                 # Agent endpoints
├── messages.py               # Message endpoints
├── tools.py                  # Tool management
├── blocks.py                 # Memory block endpoints
├── runs.py                   # Run execution endpoints
├── sources.py                # Data source endpoints
└── ...
```

### Database Models (ORM)
```
letta/orm/
├── agent.py                  # Agent ORM model
├── message.py                # Message ORM model
├── block.py                  # Memory block model
├── tool.py                   # Tool definition model
├── run.py                    # Run tracking model
└── ...                       # 40+ models
```

### LLM Provider Clients
```
letta/llm_api/
├── llm_api_client.py         # Main LLM router
├── openai_client.py          # OpenAI integration
├── anthropic_client.py       # Claude integration
├── google_vertex_client.py   # Vertex AI
└── ...                       # 10+ providers
```

### Schemas (Pydantic Models)
```
letta/schemas/
├── agent.py                  # AgentState, CreateAgent
├── message.py                # Message, LettaMessage
├── block.py                  # Block schema
├── tool.py                   # Tool schema
├── llm_config.py             # LLM configuration
└── ...                       # 75+ schema files
```

---

## Key Classes

### Agent Classes

| Class | Location | Purpose |
|-------|----------|---------|
| `BaseAgent` | `letta/agents/base.py` | Abstract base for all agents |
| `LettaAgent` | `letta/agents/letta_agent.py` | Main production agent |
| `LettaAgentV3` | `letta/agents/letta_agent_v3.py` | Latest agent implementation |
| `VoiceAgent` | `letta/agents/voice_agent.py` | Voice-enabled agent |

### Manager Classes

| Manager | Location | Key Methods |
|---------|----------|-------------|
| `AgentManager` | `letta/services/agent_manager.py` | `create_agent_async()`, `get_agent_by_id_async()` |
| `MessageManager` | `letta/services/message_manager.py` | `create_message_async()`, `get_messages()` |
| `ToolManager` | `letta/services/tool_manager.py` | `create_tool_async()`, `attach_tool()` |
| `BlockManager` | `letta/services/block_manager.py` | `create_block()`, `update_block_async()` |

### Server Classes

| Class | Location | Purpose |
|-------|----------|---------|
| `SyncServer` | `letta/server/server.py` | Main server orchestrator |
| `LettaAPI` | `letta/server/rest_api/app.py` | FastAPI application |
| `StreamingServerInterface` | `letta/server/rest_api/interface.py` | Streaming handler |

---

## Common Workflows

### Creating an Agent

**Code Path:**
```
Client → POST /v1/agents/ → agents.py:create_agent()
  → server.agent_manager.create_agent_async()
  → AgentORM.create() in database
  → Return AgentState
```

**Key Files:**
- `letta/server/rest_api/routers/v1/agents.py:create_agent()`
- `letta/services/agent_manager.py:create_agent_async()`
- `letta/orm/agent.py:Agent`

### Sending a Message

**Code Path:**
```
Client → POST /v1/agents/{id}/messages → messages.py:create_agent_message()
  → message_manager.create_message_async() (persist)
  → agent.step(messages) (execute)
  → llm_client.create_chat_completion_async() (call LLM)
  → tool_manager.execute_tool() (if tool calls)
  → message_manager.create_message_async() (save response)
  → Return LettaResponse (streaming)
```

**Key Files:**
- `letta/server/rest_api/routers/v1/messages.py:create_agent_message()`
- `letta/agents/letta_agent.py:step()`
- `letta/services/message_manager.py`

### Tool Execution

**Code Path:**
```
Agent selects tool → ToolExecutionManager.execute_tool()
  → Validate tool schema
  → Execute in sandbox (LOCAL/E2B/MODAL)
  → Return ToolExecutionResult
  → Agent processes result
```

**Key Files:**
- `letta/services/tool_manager.py:execute_tool()`
- `letta/functions/function_sets/base.py` (core tools)
- `letta/sandbox/` (execution environments)

### Memory Block Update

**Code Path:**
```
Agent calls core_memory_replace/memory tool
  → BlockManager.update_block_async()
  → Validate char limit
  → BlockORM.update() in database
  → Update in-memory AgentState
  → Return success
```

**Key Files:**
- `letta/services/block_manager.py:update_block_async()`
- `letta/functions/function_sets/base.py:memory()`
- `letta/orm/block.py:Block`

---

## Configuration

### Settings File
**Location:** `letta/settings.py`

**Key Settings:**
```python
# Database
database_engine: DatabaseChoice = SQLITE | POSTGRES
database_url: str

# LLM Provider
default_llm_provider: str = "openai"
default_llm_model: str = "gpt-4"

# Server
server_host: str = "0.0.0.0"
server_port: int = 8283

# Memory
context_window_limit: int = 8192
core_memory_limit: int = 8000

# Tools
tool_sandbox: str = "LOCAL" | "E2B" | "MODAL"
```

### Environment Variables
```bash
# LLM APIs
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_CLOUD_PROJECT=project-id

# Database
DATABASE_URL=postgresql://user:pass@localhost/letta
PGVECTOR_ENABLED=true

# Server
LETTA_SERVER_PASSWORD=optional-password
CORS_ORIGINS=http://localhost:3000

# Observability
OTEL_ENABLED=true
SENTRY_DSN=https://...
```

---

## Database Models

### Core ORM Models

| Model | File | Key Columns |
|-------|------|-------------|
| `Agent` | `letta/orm/agent.py` | `id`, `name`, `memory_config`, `llm_config` |
| `Message` | `letta/orm/message.py` | `id`, `agent_id`, `role`, `content`, `created_at` |
| `Block` | `letta/orm/block.py` | `id`, `label`, `description`, `value` |
| `Tool` | `letta/orm/tool.py` | `id`, `name`, `source_code`, `json_schema` |
| `Run` | `letta/orm/run.py` | `id`, `agent_id`, `status`, `usage_statistics` |
| `File` | `letta/orm/file.py` | `id`, `file_name`, `source_id`, `metadata` |

### Database Relationships
```
Agent (1) ──────< (N) Message
Agent (N) ─────< (N) Block (via AgentBlockLink)
Agent (N) ─────< (N) Tool (via AgentToolLink)
Agent (1) ──────< (N) Run
Source (1) ─────< (N) File
Source (1) ─────< (N) Passage (archival memory)
```

---

## API Endpoints Cheatsheet

### Agent Management
```http
GET    /v1/agents/                    # List all agents
POST   /v1/agents/                    # Create agent
GET    /v1/agents/{id}                # Get agent
PATCH  /v1/agents/{id}                # Update agent
DELETE /v1/agents/{id}                # Delete agent
```

### Messages
```http
POST   /v1/agents/{id}/messages       # Send message (streaming)
GET    /v1/agents/{id}/messages       # List messages
PATCH  /v1/agents/{id}/messages/{msg_id}  # Update message
```

### Memory (Blocks)
```http
GET    /v1/agents/{id}/core-memory/blocks           # List blocks
GET    /v1/agents/{id}/core-memory/blocks/{label}   # Get block
PATCH  /v1/agents/{id}/core-memory/blocks/{label}   # Update block
PATCH  /v1/agents/{id}/core-memory/blocks/attach/{block_id}  # Attach
```

### Tools
```http
GET    /v1/tools/                     # List all tools
POST   /v1/tools/                     # Create custom tool
GET    /v1/agents/{id}/tools          # List agent tools
PATCH  /v1/agents/{id}/tools/attach/{tool_id}    # Attach tool
```

### Files & Folders
```http
POST   /v1/folders/                   # Create folder
POST   /v1/folders/{id}/files         # Upload file
GET    /v1/agents/{id}/files          # List agent files
PATCH  /v1/agents/{id}/folders/attach/{folder_id}  # Attach folder
```

### Runs
```http
GET    /v1/runs/{run_id}              # Get run status
GET    /v1/runs/{run_id}/messages     # Get run messages
GET    /v1/runs/stream/{run_id}       # Stream run updates
```

---

## Tool Development

### Creating a Custom Tool

**1. Define the Python function:**
```python
# In your code
def my_custom_tool(query: str, limit: int = 10) -> str:
    """
    Search for information.

    Args:
        query: The search query
        limit: Maximum results to return

    Returns:
        Search results as JSON string
    """
    # Implementation
    return json.dumps({"results": [...]})
```

**2. Register via API:**
```python
from letta_client import Letta

client = Letta(token="...")

tool = client.tools.create(
    name="my_custom_tool",
    source_code=open("my_tool.py").read(),
    source_type="python"
)

# Attach to agent
client.agents.tools.attach(agent_id=agent.id, tool_id=tool.id)
```

**Built-in Tool Locations:**
- Core: `letta/functions/function_sets/base.py`
- Multi-agent: `letta/functions/function_sets/multi_agent.py`
- Files: `letta/functions/function_sets/files.py`
- Voice: `letta/functions/function_sets/voice.py`

---

## Debugging Tips

### Enable Debug Logging
```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

### Check Agent State
```python
agent_state = client.agents.get(agent_id)
print(f"Memory blocks: {agent_state.memory_blocks}")
print(f"Tools: {agent_state.tools}")
print(f"LLM config: {agent_state.llm_config}")
```

### Inspect Messages
```python
messages = client.agents.messages.list(agent_id)
for msg in messages:
    print(f"{msg.role}: {msg.content}")
```

### View Run Details
```python
run = client.runs.get(run_id)
print(f"Status: {run.status}")
print(f"Token usage: {run.usage_statistics}")
print(f"Steps: {run.num_steps}")
```

### Database Debugging (Direct Access)
```python
from letta.orm import Agent, Message
from letta.server.server import SyncServer

server = SyncServer()
session = server.get_db_session()

# Query agents
agents = session.query(Agent).all()

# Query messages
messages = session.query(Message).filter_by(agent_id="agent-123").all()
```

### OpenTelemetry Traces
```bash
# Enable OTEL
export OTEL_ENABLED=true
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318

# View traces in your OTEL collector (Jaeger, etc.)
```

---

## Code Navigation Tips

### Finding Specific Functionality

| To find... | Look in... |
|------------|-----------|
| How agents execute | `letta/agents/letta_agent.py:step()` |
| How tools are called | `letta/services/tool_manager.py:execute_tool()` |
| How memory is managed | `letta/services/block_manager.py` |
| How LLMs are called | `letta/llm_api/llm_api_client.py` |
| How streaming works | `letta/server/rest_api/interface.py` |
| Database migrations | `alembic/versions/` |
| Example usage | `examples/` directory |
| Test patterns | `tests/` directory |

### Search Tips
```bash
# Find all agent types
grep -r "class.*Agent" letta/agents/

# Find all API endpoints
grep -r "@router\." letta/server/rest_api/routers/

# Find all ORM models
grep -r "class.*Base" letta/orm/

# Find all service managers
ls letta/services/*_manager.py
```

---

## Performance Monitoring

### Key Metrics to Track
- **Token Usage**: `run.usage_statistics.total_tokens`
- **Latency**: `run.metadata.latency_ms`
- **Message Count**: `len(agent.messages)`
- **Context Window**: `agent.context_window_limit`
- **Database Queries**: Enable SQLAlchemy echo

### Optimization Tips
1. Use streaming for long responses
2. Limit archival memory search results
3. Enable connection pooling
4. Use PostgreSQL for production
5. Enable Redis caching
6. Monitor context window usage
7. Use background jobs for heavy tasks

---

## Common Error Solutions

### "Context window full"
- **Cause**: Too many messages in context
- **Solution**: Trigger summarization or increase context window limit

### "Tool execution timeout"
- **Cause**: Tool took too long
- **Solution**: Increase timeout or optimize tool code

### "Database lock"
- **Cause**: SQLite concurrent access
- **Solution**: Switch to PostgreSQL

### "Out of memory"
- **Cause**: Large embedding vectors
- **Solution**: Use Pinecone or optimize batch size

---

**Quick Links:**
- [Full Architecture Analysis](./ARCHITECTURE_ANALYSIS.md)
- [Main README](./README.md)
- [Letta Documentation](https://docs.letta.com)
