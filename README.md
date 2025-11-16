# Letta Architecture Analysis

This repository contains a comprehensive architectural analysis of the [Letta](https://github.com/letta-ai/letta) project (formerly MemGPT), an open-source platform for building stateful AI agents with advanced memory capabilities.

## What is Letta?

Letta is the platform for building stateful agents: open AI with advanced memory that can learn and self-improve over time. It implements the MemGPT LLM Operating System principles for memory management and agent orchestration.

**Key Features:**
- Stateful agents with persistent memory
- Multi-agent orchestration with shared memory
- Support for 10+ LLM providers (OpenAI, Anthropic Claude, Google Vertex, local LLMs via Ollama, etc.)
- Advanced memory hierarchy (in-context + out-of-context with embeddings)
- Comprehensive tool system with MCP (Model Context Protocol) support
- Production-ready REST API with streaming support
- Self-editing memory capabilities

## Repository Structure

```
letta-grok/
├── README.md                          # This file - quick overview
├── ARCHITECTURE_ANALYSIS.md           # Comprehensive 13-section architecture analysis
└── letta-repo/                        # Cloned Letta source code
    ├── letta/                         # Main Python package
    ├── tests/                         # Test suite
    ├── examples/                      # Usage examples
    └── ...
```

## Documentation

### [📖 Read the Complete Architecture Analysis](./ARCHITECTURE_ANALYSIS.md)

The comprehensive analysis document covers:

1. **Executive Summary** - High-level overview
2. **Directory Structure** - Complete codebase organization
3. **Core Modules** - 28 modules with detailed responsibilities
4. **Tech Stack** - Frameworks, databases, dependencies
5. **Architectural Patterns** - Design decisions and patterns
6. **Key Components** - Entry points and workflows
7. **API Structure** - 70+ REST endpoints reference
8. **Data Flow Examples** - Request/response flows
9. **Memory Management** - Advanced hierarchical memory system
10. **Security Architecture** - Auth, sandboxing, isolation
11. **Performance** - Scaling and optimization strategies
12. **Extensibility Points** - How to extend Letta
13. **Dependencies** - External services and integrations

## Quick Architecture Overview

### System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Applications                       │
│         (Python SDK, TypeScript SDK, ADE UI)                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   FastAPI REST API Layer                     │
│  (70+ endpoints, streaming, auth, middleware)               │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                 Service Manager Layer                        │
│  AgentManager │ MessageManager │ ToolManager │ BlockManager │
│  (Business logic, validation, orchestration)                │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Agents    │  │  Database   │  │ LLM Clients │
│  (7 types)  │  │ (ORM/SQL)   │  │ (10+ APIs)  │
└─────────────┘  └─────────────┘  └─────────────┘
```

### Core Components

- **Agents (7 types)**: LettaAgent, LettaAgentV2/V3, VoiceAgent, EphemeralAgent
- **Services (30+ managers)**: AgentManager, MessageManager, ToolManager, BlockManager, etc.
- **Database Layer**: SQLAlchemy ORM with SQLite/PostgreSQL/Pinecone support
- **LLM Integration**: 50+ provider implementations (OpenAI, Claude, Gemini, local LLMs)
- **Tool System**: Built-in tools, custom Python tools, MCP tools, sandboxed execution
- **Memory System**: Hierarchical (in-context blocks + out-of-context vector search)

### Tech Stack

| Layer | Technologies |
|-------|-------------|
| API | FastAPI 0.115+, Uvicorn, Server-Sent Events (SSE) |
| Database | SQLAlchemy 2.0+, SQLite, PostgreSQL, Pinecone |
| LLMs | OpenAI, Anthropic, Google Vertex, Ollama, vLLM |
| Tools | MCP CLI, LlamaIndex, custom Python |
| Jobs | Temporal.io, APScheduler |
| Observability | OpenTelemetry, Sentry |

## Key Architectural Features

### 1. Advanced Memory Management
- **Hierarchical Memory**: In-context (fast access) + out-of-context (vector search)
- **Self-Editing**: Agents update their own memory using tools
- **Auto-Summarization**: Context automatically compressed when full
- **Shared Blocks**: Memory blocks can be shared across multiple agents

### 2. Multi-Agent Architecture
- Agents can communicate directly via message passing
- Tag-based group messaging for coordinated workflows
- Supervisor/worker patterns for orchestration
- Shared memory blocks for collaboration

### 3. Comprehensive Tool System
- **Built-in Tools**: `send_message`, `memory`, `web_search`, `run_code`
- **Custom Tools**: Python functions registered as tools
- **MCP Tools**: Model Context Protocol integration
- **Sandboxed Execution**: LOCAL, E2B, MODAL isolation

### 4. Production-Ready
- Fully async architecture
- Token-by-token streaming responses
- Database migration system (100+ Alembic migrations)
- Multi-database support with connection pooling
- OpenTelemetry observability

## Getting Started with Letta

### Installation

```bash
# Install from PyPI
pip install letta-client

# Or from source
cd letta-repo
uv sync --all-extras
uv run letta server
```

### Simple Example

```python
from letta_client import Letta
import os

# Connect to Letta Cloud
client = Letta(token=os.getenv("LETTA_API_KEY"))

# Create an agent
agent = client.agents.create(
    model="openai/gpt-4.1",
    memory_blocks=[
        {"label": "human", "value": "User prefers concise answers"},
        {"label": "persona", "value": "I am a helpful AI assistant"}
    ]
)

# Send a message
response = client.agents.messages.create(
    agent_id=agent.id,
    messages=[{"role": "user", "content": "Hello!"}]
)
```

## Exploration Paths

Depending on your interest, start exploring:

### For Application Developers
- Review SDK examples in `letta-repo/examples/`
- Check REST API endpoints in the analysis (Section 6)
- Understand memory blocks and agent creation (Section 2)

### For Contributors
- Study the service manager pattern (Section 4)
- Explore the agent execution pipeline (Section 5)
- Review the database ORM models in `letta-repo/letta/orm/`

### For System Architects
- Analyze the multi-layer architecture (Section 1-2)
- Study scaling considerations (Section 10)
- Review observability setup (Section 11)

### For AI/ML Engineers
- Understand the memory hierarchy (Section 8.1)
- Study LLM provider integration (Section 2.5)
- Explore tool execution sandboxing (Section 4.5)

## Resources

- **Official Docs**: https://docs.letta.com
- **GitHub Repo**: https://github.com/letta-ai/letta
- **Discord Community**: https://discord.gg/letta
- **Letta Cloud**: https://app.letta.com

## Contributing to Analysis

This analysis is maintained as part of understanding the Letta architecture. If you find inaccuracies or want to add sections:

1. Review the cloned source in `letta-repo/`
2. Update `ARCHITECTURE_ANALYSIS.md`
3. Submit improvements via pull request

## License

The Letta project is licensed under the Apache 2.0 License. This analysis is provided for educational purposes.

---

**Last Updated**: November 16, 2025
**Letta Version Analyzed**: Main branch (latest as of analysis date)
**Analysis Author**: Claude Code Agent
