# Database & ORM Layer - Deep Dive

Database architecture, ORM models, and migrations with code references.

## Overview

Letta uses SQLAlchemy 2.0+ with support for SQLite, PostgreSQL, and Pinecone for vector storage.

**Primary Files:**
- [`letta/orm/`](../letta-repo/letta/orm/) - ORM models (40+ files)
- [`alembic/versions/`](../letta-repo/alembic/versions/) - Database migrations (100+ files)
- [`letta/server/db.py`](../letta-repo/letta/server/db.py) - Database registry

---

## Database Support

### Supported Databases

1. **SQLite** - Default, local development
   - With `sqlite-vec` extension for vector search
2. **PostgreSQL** - Production
   - With `pgvector` extension for embeddings
3. **Pinecone** - Vector database (optional)

**Configuration:** `settings.database_url`

```python
# SQLite (default)
DATABASE_URL=sqlite:///~/.letta/letta.db

# PostgreSQL
DATABASE_URL=postgresql://user:pass@localhost/letta

# With pgvector
DATABASE_URL=postgresql://user:pass@localhost/letta?options=-c%20search_path=public
```

---

## Core ORM Models

### Agent Model

**Location:** [`letta/orm/agent.py`](../letta-repo/letta/orm/agent.py)

```python
class Agent(SQLModel, table=True):
    """Agent ORM model."""

    __tablename__ = "agents"

    # Primary key
    id: str = Field(primary_key=True)

    # Configuration
    name: str
    system: str                              # System prompt
    llm_config: Dict                         # LLM configuration (JSON)
    embedding_config: Dict                   # Embedding config (JSON)

    # Relationships
    tools: List["Tool"] = Relationship(      # Many-to-many via AgentsTool
        back_populates="agents",
        link_model=AgentsTool
    )
    memory_blocks: List["Block"] = Relationship(  # Many-to-many
        back_populates="agents",
        link_model=BlocksAgents
    )
    messages: List["Message"] = Relationship(     # One-to-many
        back_populates="agent"
    )

    # Metadata
    created_at: datetime
    updated_at: datetime
    organization_id: str
```

### Message Model

**Location:** [`letta/orm/message.py`](../letta-repo/letta/orm/message.py)

```python
class Message(SQLModel, table=True):
    """Message ORM model."""

    __tablename__ = "messages"

    id: str = Field(primary_key=True)
    agent_id: str = Field(foreign_key="agents.id")

    # Content
    role: str                                # user, assistant, tool
    content: List[Dict]                      # JSON content blocks
    tool_calls: Optional[List[Dict]]         # Tool invocations

    # Relationships
    agent: "Agent" = Relationship(back_populates="messages")

    # Metadata
    created_at: datetime
    step_id: Optional[str]                   # Links to Step
```

### Block Model

**Location:** [`letta/orm/block.py`](../letta-repo/letta/orm/block.py)

```python
class Block(SQLModel, table=True):
    """Memory block ORM model."""

    __tablename__ = "blocks"

    id: str = Field(primary_key=True)
    label: str                               # "human", "persona", etc.
    value: str                               # Block content
    limit: int                               # Character limit

    # Relationships
    agents: List["Agent"] = Relationship(
        back_populates="memory_blocks",
        link_model=BlocksAgents
    )
```

---

## Relationships

### Many-to-Many: Agents ↔ Tools

**Junction Table:** [`letta/orm/agents_tools.py`](../letta-repo/letta/orm/agents_tools.py)

```python
class AgentsTool(SQLModel, table=True):
    """Junction table for agents and tools."""

    __tablename__ = "agents_tools"

    agent_id: str = Field(foreign_key="agents.id", primary_key=True)
    tool_id: str = Field(foreign_key="tools.id", primary_key=True)
```

### Many-to-Many: Agents ↔ Blocks

**Junction Table:** [`letta/orm/blocks_agents.py`](../letta-repo/letta/orm/blocks_agents.py)

```python
class BlocksAgents(SQLModel, table=True):
    """Junction table for blocks and agents."""

    __tablename__ = "blocks_agents"

    block_id: str = Field(foreign_key="blocks.id", primary_key=True)
    agent_id: str = Field(foreign_key="agents.id", primary_key=True)
```

---

## Database Migrations

### Alembic Setup

**Location:** [`alembic/`](../letta-repo/alembic/)

Letta uses Alembic for schema versioning with 100+ migration files.

**Migration Structure:**
```
alembic/
├── versions/
│   ├── 001_initial_schema.py
│   ├── 002_add_llm_config.py
│   ├── 003_add_embedding_config.py
│   └── ...100+ migrations
├── env.py                    # Migration environment
└── script.py.mako            # Migration template
```

### Running Migrations

```bash
# Upgrade to latest
alembic upgrade head

# Downgrade one version
alembic downgrade -1

# Create new migration
alembic revision --autogenerate -m "Add new field"
```

---

## Vector Search

### pgvector (PostgreSQL)

**Location:** [`letta/orm/passage.py`](../letta-repo/letta/orm/passage.py)

```python
class ArchivalPassage(SQLModel, table=True):
    """Archival memory passage with embedding."""

    __tablename__ = "archival_passages"

    id: str = Field(primary_key=True)
    text: str                                # Passage content
    embedding: List[float]                   # Vector (1536 dims)
    archive_id: str

# Vector similarity search
query = select(ArchivalPassage).where(
    ArchivalPassage.archive_id == archive_id
).order_by(
    ArchivalPassage.embedding.cosine_distance(query_embedding)
).limit(top_k)
```

### sqlite-vec (SQLite)

For local development, Letta uses `sqlite-vec` extension.

```sql
SELECT text, 
       1 - vector_distance(embedding, ?) AS similarity
FROM archival_passages
WHERE archive_id = ?
ORDER BY similarity DESC
LIMIT ?
```

---

## Database Registry

**Location:** [`letta/server/db.py`](../letta-repo/letta/server/db.py)

```python
class DatabaseRegistry:
    """
    Central registry for database sessions.

    Handles connection pooling and session lifecycle.
    """

    def __init__(self):
        self.engine = create_async_engine(
            settings.database_url,
            pool_size=10,
            max_overflow=20,
        )

    @asynccontextmanager
    async def async_session(self) -> AsyncSession:
        """
        Context manager for database sessions.

        Usage:
            async with db_registry.async_session() as session:
                result = await session.execute(query)
        """
        async with AsyncSession(self.engine) as session:
            try:
                yield session
            except Exception:
                await session.rollback()
                raise
            finally:
                await session.close()

# Global registry
db_registry = DatabaseRegistry()
```

---

## Query Patterns

### Async Queries

```python
# Read single record
async with db_registry.async_session() as session:
    stmt = select(Agent).where(Agent.id == agent_id)
    result = await session.execute(stmt)
    agent = result.scalar_one_or_none()

# Read with relationships
stmt = (
    select(Agent)
    .where(Agent.id == agent_id)
    .options(
        selectinload(Agent.tools),
        selectinload(Agent.memory_blocks)
    )
)

# Create
agent = Agent(id="agent-123", name="Test")
session.add(agent)
await session.commit()

# Update
agent.name = "Updated"
await session.commit()

# Delete
await session.delete(agent)
await session.commit()
```

---

## See Also

- [Agent Execution](./agent-execution.md) - How database integrates with execution
- [Memory Management](./memory-management.md) - Block and passage storage
