# Memory Management System - Deep Dive

This document provides a detailed analysis of Letta's hierarchical memory system with specific code references and line numbers.

## Table of Contents
- [Overview](#overview)
- [Memory Architecture](#memory-architecture)
- [Core Memory (In-Context)](#core-memory-in-context)
- [Archival Memory (Out-of-Context)](#archival-memory-out-of-context)
- [Memory Tools](#memory-tools)
- [Memory Updates Flow](#memory-updates-flow)
- [Summarization System](#summarization-system)
- [Code Examples](#code-examples)

---

## Overview

Letta implements a **hierarchical memory system** inspired by the MemGPT LLM Operating System paper. This allows agents to maintain long-term memory beyond the context window limit.

**Key Insight:** Like a traditional OS with RAM and disk storage, Letta has:
- **In-Context Memory (RAM)**: Fast, limited, directly in LLM context window
- **Out-of-Context Memory (Disk)**: Slow, unlimited, retrieved via semantic search

**Primary Files:**
- [`letta/schemas/memory.py`](../letta-repo/letta/schemas/memory.py) - Memory class (467 lines)
- [`letta/schemas/block.py`](../letta-repo/letta/schemas/block.py) - Block definitions (208 lines)
- [`letta/services/block_manager.py`](../letta-repo/letta/services/block_manager.py) - Block CRUD operations
- [`letta/services/passage_manager.py`](../letta-repo/letta/services/passage_manager.py) - Archival memory manager

---

## Memory Architecture

### Three-Tier System

```
┌─────────────────────────────────────────────────────────┐
│           In-Context Memory (Context Window)            │
├─────────────────────────────────────────────────────────┤
│  1. System Prompt                                       │
│     ├─ Base instructions                               │
│     └─ Memory blocks (editable by agent)               │
│  2. Recent Messages                                     │
│     └─ Last N messages that fit in context             │
│  3. Tool Definitions                                    │
│     └─ Available function schemas                      │
└─────────────────────────────────────────────────────────┘
                         ↕
              (Context window full?)
                         ↕
┌─────────────────────────────────────────────────────────┐
│        Out-of-Context Memory (Database + Vectors)       │
├─────────────────────────────────────────────────────────┤
│  1. Full Message History                                │
│     └─ All messages ever sent (in DB)                  │
│  2. Archival Memory Passages                            │
│     ├─ Manually added facts/information                │
│     ├─ Summarized old conversations                     │
│     └─ Vector embeddings for search                    │
│  3. Conversation Summary                                │
│     └─ Auto-generated summary of evicted messages      │
└─────────────────────────────────────────────────────────┘
```

---

## Core Memory (In-Context)

### Memory Class

**Location:** [`letta/schemas/memory.py:56-364`](../letta-repo/letta/schemas/memory.py#L56-L364)

```python
class Memory(BaseModel, validate_assignment=True):
    """
    Represents the in-context memory (i.e. Core memory) of the agent.
    This includes both the Block objects (labelled by sections), as well
    as tools to edit the blocks.
    """

    agent_type: Optional[Union["AgentType", str]] = Field(...)  # Line 63
    blocks: List[Block] = Field(...)                             # Line 64 - Core memory blocks
    file_blocks: List[FileBlock] = Field(...)                    # Line 65 - File contents

    def compile(                                                 # Line 271
        self,
        tool_usage_rules=None,
        sources=None,
        max_files_open=None,
        llm_config=None
    ) -> str:
        """
        Efficiently render memory, tool rules, and sources into a
        prompt string.
        """
```

**Key Methods:**
- `compile()` - Renders blocks to string for system prompt (line 271-308)
- `get_block(label)` - Retrieve block by label (line 333-340)
- `update_block_value(label, value)` - Update block content (line 354-363)

### Block Schema

**Location:** [`letta/schemas/block.py:13-80`](../letta-repo/letta/schemas/block.py#L13-L80)

```python
class BaseBlock(LettaBase, validate_assignment=True):
    """Base block of the LLM context"""

    # Core fields
    value: str = Field(..., description="Value of the block.")         # Line 19
    limit: int = Field(
        CORE_MEMORY_BLOCK_CHAR_LIMIT,                                  # Line 20 - Default: 8000 chars
        description="Character limit of the block."
    )

    # Metadata
    label: Optional[str] = Field(                                       # Line 33
        None,
        description="Label of the block (e.g. 'human', 'persona')"
    )
    description: Optional[str] = Field(...)                             # Line 39
    read_only: bool = Field(False, ...)                                 # Line 36

    @model_validator(mode="before")
    @classmethod
    def verify_char_limit(cls, data: Any) -> Any:                       # Line 51-72
        """Validate the character limit before model instantiation."""
        if isinstance(data, dict):
            limit = data.get("limit")
            value = data.get("value")

            if limit is not None and value is not None:
                if len(value) > limit:                                  # Line 68
                    error_msg = (
                        f"Edit failed: Exceeds {limit} character limit "
                        f"(requested {len(value)})"
                    )
                    raise ValueError(error_msg)

        return data
```

**Character Limit Enforcement:**
- Default limit: 8,000 characters (constant in `letta/constants.py`)
- Validated on creation and update
- Prevents context window overflow

### Default Memory Blocks

**Location:** [`letta/schemas/block.py:121-135`](../letta-repo/letta/schemas/block.py#L121-L135)

```python
class Human(Block):
    """Human block of the LLM context"""
    label: str = "human"                                                # Line 124
    description: Optional[str] = Field(
        DEFAULT_HUMAN_BLOCK_DESCRIPTION,
        description="Description of the block."
    )

class Persona(Block):
    """Persona block of the LLM context"""
    label: str = "persona"                                              # Line 131
    description: Optional[str] = Field(
        DEFAULT_PERSONA_BLOCK_DESCRIPTION,
        description="Description of the block."
    )

DEFAULT_BLOCKS = [Human(value=""), Persona(value="")]                  # Line 135
```

**Standard Blocks:**
1. **`human`**: Information about the user
2. **`persona`**: Agent's personality and role

**Example Rendered in System Prompt:**
```xml
<memory_blocks>
The following memory blocks are currently engaged in your core memory unit:

<human>
<description>
Information about the human you are conversing with.
</description>
<metadata>
- chars_current=85
- chars_limit=8000
</metadata>
<value>
Name: John Doe
Occupation: Software Engineer
Preferences: Likes concise answers
</value>
</human>

<persona>
<description>
Your persona and role in this conversation.
</description>
<metadata>
- chars_current=62
- chars_limit=8000
</metadata>
<value>
I am a helpful AI assistant focused on technical accuracy.
</value>
</persona>

</memory_blocks>
```

### Memory Rendering

**Location:** [`letta/schemas/memory.py:110-169`](../letta-repo/letta/schemas/memory.py#L110-L169)

```python
@trace_method
def _render_memory_blocks_standard(self, s: StringIO):                 # Line 110
    """Render memory blocks in standard XML format."""
    if len(self.blocks) == 0:
        s.write("")
        return

    s.write("<memory_blocks>\n")                                       # Line 116
    s.write("The following memory blocks are currently engaged ")
    s.write("in your core memory unit:\n\n")

    for idx, block in enumerate(self.blocks):                          # Line 117
        label = block.label or "block"
        value = block.value or ""
        desc = block.description or ""
        chars_current = len(value)
        limit = block.limit if block.limit is not None else 0

        s.write(f"<{label}>\n")                                        # Line 124
        s.write("<description>\n")
        s.write(f"{desc}\n")
        s.write("</description>\n")
        s.write("<metadata>")                                          # Line 128
        if getattr(block, "read_only", False):
            s.write("\n- read_only=true")
        s.write(f"\n- chars_current={chars_current}")                  # Line 131
        s.write(f"\n- chars_limit={limit}\n")
        s.write("</metadata>\n")
        s.write("<value>\n")
        s.write(f"{value}\n")                                          # Line 135
        s.write("</value>\n")
        s.write(f"</{label}>\n")

    s.write("\n</memory_blocks>")                                      # Line 140
```

**Line-Numbered Variant (Anthropic Models):**

**Location:** [`letta/schemas/memory.py:142-169`](../letta-repo/letta/schemas/memory.py#L142-L169)

```python
def _render_memory_blocks_line_numbered(self, s: StringIO):            # Line 142
    """
    Render with line numbers (used for Anthropic models to enable
    precise str_replace operations).
    """
    # ... similar structure ...
    s.write("<value>\n")
    if value:
        for i, line in enumerate(value.split("\n"), start=1):          # Line 163
            s.write(f"{i}→ {line}\n")                                  # Line 164
    s.write("</value>\n")
```

**Example Line-Numbered Output:**
```xml
<value>
1→ Name: John Doe
2→ Occupation: Software Engineer
3→ Preferences: Likes concise answers
</value>
```

---

## Archival Memory (Out-of-Context)

### Passage Schema

**Location:** [`letta/orm/passage.py`](../letta-repo/letta/orm/passage.py)

Archival memory stores "passages" - chunks of text with embeddings for semantic search.

```python
class ArchivalPassage(SQLModel, table=True):
    """Represents a passage in archival memory."""
    id: str                       # Unique ID
    text: str                     # Passage content
    embedding: List[float]        # Vector embedding (1536 dims for OpenAI)
    archive_id: str               # Associated archive
    created_at: datetime          # Timestamp
    tags: List[str]               # Optional tags
```

### PassageManager

**Location:** [`letta/services/passage_manager.py:51-100`](../letta-repo/letta/services/passage_manager.py#L51-L100)

```python
class PassageManager:
    """Manager class to handle business logic related to Passages."""

    def __init__(self):
        self.archive_manager = ArchiveManager()                        # Line 55

    async def create_agent_passage_async(                              # ~Line 150+
        self,
        agent_id: str,
        text: str,
        tags: List[str],
        actor: PydanticUser,
    ) -> PydanticPassage:
        """
        Create a new passage in agent's archival memory.

        Process:
        1. Generate embedding for text
        2. Store passage in database with embedding
        3. Enable semantic search retrieval
        """
        # Generate embedding
        embedding = await self._get_embedding_async(text)

        # Create passage
        passage = ArchivalPassage(
            id=generate_id(),
            text=text,
            embedding=embedding,
            archive_id=agent.default_archive_id,
            created_at=utcnow(),
            tags=tags,
        )

        # Persist to database
        await session.add(passage)
        await session.commit()

        return passage.to_pydantic()

    async def search_agent_passages_async(                             # ~Line 250+
        self,
        agent_id: str,
        query: str,
        top_k: int = 5,
        actor: PydanticUser,
    ) -> List[PydanticPassage]:
        """
        Semantic search in agent's archival memory.

        Process:
        1. Generate embedding for query
        2. Vector similarity search in database
        3. Return top-k most similar passages
        """
        # Generate query embedding
        query_embedding = await self._get_embedding_async(query)

        # Vector similarity search (cosine similarity)
        results = await self._vector_search(
            archive_id=agent.default_archive_id,
            query_embedding=query_embedding,
            top_k=top_k,
        )

        return results
```

### Embedding Generation

**Location:** [`letta/services/passage_manager.py:34-48`](../letta-repo/letta/services/passage_manager.py#L34-L48)

```python
@lru_cache(maxsize=8192)                                               # Line 33
def get_openai_embedding(text: str, model: str, endpoint: str) -> List[float]:
    """
    Generate embedding using OpenAI API.
    Cached for performance (same text = same embedding).
    """
    client = OpenAI(api_key=..., base_url=endpoint, max_retries=0)   # Line 37
    response = client.embeddings.create(input=text, model=model)       # Line 38
    return response.data[0].embedding                                  # Line 39

@async_redis_cache(key_func=lambda text, model, endpoint: f"{model}:{endpoint}:{text}")
async def get_openai_embedding_async(                                 # Line 43
    text: str,
    model: str,
    endpoint: str
) -> list[float]:
    """Async version with Redis caching."""
    client = AsyncOpenAI(...)
    response = await client.embeddings.create(input=text, model=model) # Line 47
    return response.data[0].embedding                                  # Line 48
```

**Performance Optimizations:**
- LRU cache for sync calls (8192 entries)
- Redis cache for async calls (distributed caching)
- Embeddings are expensive - caching critical

---

## Memory Tools

### Unified Memory Tool

**Location:** [`letta/functions/function_sets/base.py:9-67`](../letta-repo/letta/functions/function_sets/base.py#L9-L67)

```python
def memory(
    agent_state: "AgentState",
    command: str,                                                      # Line 11
    path: Optional[str] = None,                                        # Line 12
    file_text: Optional[str] = None,                                   # Line 13
    description: Optional[str] = None,                                 # Line 14
    old_str: Optional[str] = None,                                     # Line 15
    new_str: Optional[str] = None,                                     # Line 16
    insert_line: Optional[int] = None,                                 # Line 17
    insert_text: Optional[str] = None,                                 # Line 18
    old_path: Optional[str] = None,                                    # Line 19
    new_path: Optional[str] = None,                                    # Line 20
) -> Optional[str]:
    """
    Memory management tool with various sub-commands for memory
    block operations.

    Args:
        command (str): The sub-command to execute. Supported commands:
            - "create": Create a new memory block                      # Line 27
            - "str_replace": Replace text in a memory block            # Line 28
            - "insert": Insert text at a specific line                 # Line 29
            - "delete": Delete a memory block                          # Line 30
            - "rename": Rename a memory block                          # Line 31

    Examples:
        # Replace text in a memory block
        memory(                                                        # Line 46
            agent_state,
            "str_replace",
            path="/memories/user_preferences",
            old_str="theme: dark",
            new_str="theme: light"
        )

        # Insert text at line 5
        memory(agent_state, "insert", path="/memories/notes",
               insert_line=5, insert_text="New note here")            # Line 50

        # Create a memory block
        memory(                                                        # Line 62
            agent_state,
            "create",
            path="/memories/coding_preferences",
            description="The user's coding preferences.",
            file_text="The user seems to add type hints..."
        )
    """
    raise NotImplementedError(                                         # Line 67
        "This should never be invoked directly. Contact Letta if you "
        "see this error message."
    )
```

**Important Note:**
This is a **declaration-only** tool. The actual implementation is handled by:
- [`letta/services/tool_executor/tool_execution_manager.py`](../letta-repo/letta/services/tool_executor/tool_execution_manager.py)

The LLM sees this function signature, but execution is intercepted and handled specially.

### Legacy Memory Tools

**Also in:** [`letta/functions/function_sets/base.py`](../letta-repo/letta/functions/function_sets/base.py)

These are older tools still supported for backward compatibility:

```python
def core_memory_append(
    agent_state: "AgentState",
    label: str,
    content: str
) -> Optional[str]:
    """
    Append to the contents of core memory.

    This is a legacy tool - prefer using memory() instead.
    """

def core_memory_replace(
    agent_state: "AgentState",
    label: str,
    old_content: str,
    new_content: str
) -> Optional[str]:
    """
    Replace the contents of core memory.

    This is a legacy tool - prefer using memory() instead.
    """
```

### Archival Memory Tools

```python
def archival_memory_insert(
    agent_state: "AgentState",
    content: str,
    tags: Optional[List[str]] = None
) -> Optional[str]:
    """
    Add a memory passage to archival memory for long-term storage.
    """

def archival_memory_search(
    agent_state: "AgentState",
    query: str,
    top_k: int = 5
) -> Optional[str]:
    """
    Search archival memory using semantic similarity.
    Returns top-k most relevant passages.
    """
```

---

## Memory Updates Flow

### Step-by-Step: Updating Memory

**Entry Point:** Agent calls `memory()` tool from LLM response

**Execution Flow:**

```python
# 1. LLM generates tool call
{
  "tool_calls": [
    {
      "function": {
        "name": "memory",
        "arguments": {
          "command": "str_replace",
          "path": "/memories/human",
          "old_str": "Name: unknown",
          "new_str": "Name: John Doe"
        }
      }
    }
  ]
}

# 2. ToolExecutionManager intercepts
# Location: letta/services/tool_executor/tool_execution_manager.py
async def execute_tool_async(
    self,
    agent_state: AgentState,
    tool_name: str,
    tool_args: dict,
    actor: User,
) -> ToolExecutionResult:
    """Execute tool and return result."""

    if tool_name == "memory":
        # Special handling for memory tool
        return await self._execute_memory_tool(
            agent_state, tool_args, actor
        )

# 3. Extract block label from path
path = "/memories/human"  # or just "human" for core blocks
block_label = path.split("/")[-1]  # "human"

# 4. Get current block
# Location: letta/schemas/memory.py:333-340
current_block = agent_state.memory.get_block(block_label)

# 5. Perform string replacement
old_value = current_block.value
new_value = old_value.replace(
    tool_args["old_str"],
    tool_args["new_str"]
)

# 6. Validate character limit
# Location: letta/schemas/block.py:51-72
if len(new_value) > current_block.limit:
    raise ValueError(
        f"Edit failed: Exceeds {current_block.limit} character "
        f"limit (requested {len(new_value)})"
    )

# 7. Update block in database
# Location: letta/services/block_manager.py:139-166
async def update_block_async(
    self,
    block_id: str,
    block_update: BlockUpdate,
    actor: User
) -> Block:
    async with db_registry.async_session() as session:
        block = await BlockModel.read_async(
            db_session=session,
            identifier=block_id,
            actor=actor
        )

        # Validate limit constraints
        validate_block_limit_constraint(update_data, block)          # Line 146

        # Update fields
        for key, value in update_data.items():
            setattr(block, key, value)                               # Line 149

        # Persist to database
        await block.update_async(
            db_session=session,
            actor=actor
        )

        # Return updated block
        return block.to_pydantic()

# 8. Update in-memory agent state
agent_state.memory.update_block_value(block_label, new_value)

# 9. Trigger system prompt rebuild
# Location: letta/agents/base_agent.py:86-193
# This happens in the next step of the execution loop
in_context_messages = await self._rebuild_memory_async(
    in_context_messages=in_context_messages,
    agent_state=agent_state,
    tool_rules_solver=tool_rules_solver,
)

# 10. Next LLM call sees updated memory
# The system message now contains the new memory value
```

### Block Manager Operations

**Location:** [`letta/services/block_manager.py:84-166`](../letta-repo/letta/services/block_manager.py#L84-L166)

**Create Block:**
```python
@trace_method
async def create_or_update_block_async(                               # Line 89
    self,
    block: PydanticBlock,
    actor: PydanticUser
) -> PydanticBlock:
    """Create a new block based on the Block schema."""
    db_block = await self.get_block_by_id_async(block.id, actor)      # Line 91
    if db_block:
        # Block exists, update it
        update_data = BlockUpdate(**block.model_dump(...))
        return await self.update_block_async(block.id, update_data, actor)
    else:
        # Create new block
        async with db_registry.async_session() as session:
            data = block.model_dump(to_orm=True, exclude_none=True)
            validate_block_creation(data)                             # Line 99
            block = BlockModel(**data, organization_id=actor.organization_id)
            await block.create_async(session, actor=actor, ...)       # Line 101
            await session.commit()                                    # Line 103
            return block.to_pydantic()
```

**Update Block:**
```python
@trace_method
async def update_block_async(                                         # Line 139
    self,
    block_id: str,
    block_update: BlockUpdate,
    actor: PydanticUser
) -> PydanticBlock:
    """Update a block by its ID with the given BlockUpdate object."""
    async with db_registry.async_session() as session:
        block = await BlockModel.read_async(...)                      # Line 142
        update_data = block_update.model_dump(...)                    # Line 143

        # Validate limit constraints before updating
        validate_block_limit_constraint(update_data, block)           # Line 146

        for key, value in update_data.items():
            setattr(block, key, value)                                # Line 148-149

        # Add to block history (audit trail)
        block_history = BlockHistory(...)                             # ~Line 152
        await block_history.create_async(session, actor, ...)

        await block.update_async(session, actor, ...)
        await session.commit()
        return block.to_pydantic()
```

**Validation:**
```python
def validate_block_limit_constraint(                                  # Line 28
    update_data: dict,
    existing_block: BlockModel
) -> None:
    """
    Validates that block limit constraints are satisfied when
    updating a block.

    Rules:
    - If limit is being updated, it must be >= the length of the value
    - If value is being updated, its length must not exceed the limit
    """
    if "limit" in update_data:
        value_to_check = update_data.get("value", existing_block.value) # Line 46
        limit_to_check = update_data["limit"]
        if value_to_check and limit_to_check < len(value_to_check):   # Line 48
            raise LettaInvalidArgumentError(
                f"Limit ({limit_to_check}) cannot be less than "
                f"current value length ({len(value_to_check)} chars)"
            )
    elif "value" in update_data and existing_block.limit:
        if len(update_data["value"]) > existing_block.limit:           # Line 55
            raise LettaInvalidArgumentError(
                f"Value length ({len(update_data['value'])} chars) "
                f"exceeds block limit ({existing_block.limit} chars)"
            )
```

---

## Summarization System

### When Summarization Triggers

**Context Window Pressure Detection:**

**Location:** [`letta/agents/letta_agent.py`](../letta-repo/letta/agents/letta_agent.py) (various locations)

```python
# Check if context window is approaching limit
if current_tokens > (context_window_limit * 0.9):                      # ~90% full
    # Trigger summarization
    await self._handle_context_window_pressure()
```

### Summarization Process

**Location:** [`letta/services/summarizer/summarizer.py`](../letta-repo/letta/services/summarizer/summarizer.py)

```python
class Summarizer:
    """
    Handles automatic summarization of conversation history
    when context window fills up.
    """

    async def summarize_messages_async(
        self,
        agent_state: AgentState,
        messages_to_summarize: List[Message],
        actor: User,
    ) -> str:
        """
        Summarize a set of messages using an ephemeral agent.

        Process:
        1. Create EphemeralSummaryAgent (temporary agent)
        2. Pass messages to be summarized
        3. LLM generates concise summary
        4. Save summary to 'conversation_summary' block
        5. Delete old messages from context
        6. Keep summary in context instead
        """

        # Create summarizer agent
        summarizer_agent = EphemeralSummaryAgent(...)

        # Generate summary
        summary = await summarizer_agent.step(
            input_messages=[
                MessageCreate(
                    role="user",
                    content=f"Summarize: {messages_to_summarize}"
                )
            ]
        )

        # Update summary block
        await block_manager.update_block_async(
            block_id=summary_block.id,
            block_update=BlockUpdate(value=summary),
            actor=actor,
        )

        return summary
```

**Summarization Modes:**

**Location:** [`letta/services/summarizer/enums.py`](../letta-repo/letta/services/summarizer/enums.py)

```python
class SummarizationMode(str, Enum):
    STATIC_BUFFER = "static_buffer"       # Fixed message window
    PARTIAL_EVICT = "partial_evict"       # Evict percentage
    DISABLED = "disabled"                 # No summarization
```

**Configuration:**

```python
# letta/settings.py
class SummarizerSettings:
    mode: SummarizationMode = SummarizationMode.STATIC_BUFFER
    message_buffer_limit: int = 50        # Max messages before summarize
    message_buffer_min: int = 20          # Keep this many recent
    enable_summarization: bool = True
    partial_evict_summarizer_percentage: float = 0.5  # 50%
```

---

## Code Examples

### Example 1: Agent Updates Its Memory

```python
# User says: "My name is Sarah"

# Agent execution step
agent_response = await agent.step(
    input_messages=[
        MessageCreate(role="user", content="My name is Sarah")
    ]
)

# LLM thinks and calls memory tool
{
  "role": "assistant",
  "content": "I should remember the user's name.",
  "tool_calls": [
    {
      "function": {
        "name": "memory",
        "arguments": {
          "command": "str_replace",
          "path": "human",
          "old_str": "Name: unknown",
          "new_str": "Name: Sarah"
        }
      }
    }
  ]
}

# Memory is updated in database
# Next message sees updated memory

# Agent responds
{
  "tool_calls": [
    {
      "function": {
        "name": "send_message",
        "arguments": {
          "message": "Nice to meet you, Sarah!"
        }
      }
    }
  ]
}
```

### Example 2: Searching Archival Memory

```python
# User asks: "What did we discuss about Python last week?"

# Agent calls archival_memory_search
{
  "tool_calls": [
    {
      "function": {
        "name": "archival_memory_search",
        "arguments": {
          "query": "Python discussion",
          "top_k": 5
        }
      }
    }
  ]
}

# PassageManager performs vector search
query_embedding = await get_openai_embedding_async(
    text="Python discussion",
    model="text-embedding-3-small",
    endpoint="https://api.openai.com/v1"
)

results = await db.query(
    """
    SELECT text, created_at,
           1 - (embedding <=> $1) AS similarity
    FROM archival_passages
    WHERE archive_id = $2
    ORDER BY similarity DESC
    LIMIT 5
    """
)

# Results returned to agent
[
    {
        "text": "User mentioned learning Python decorators...",
        "timestamp": "2025-11-10 14:30:00",
        "similarity": 0.89
    },
    {
        "text": "We discussed Python async/await patterns...",
        "timestamp": "2025-11-09 10:15:00",
        "similarity": 0.82
    }
]

# Agent uses results to formulate response
{
  "tool_calls": [
    {
      "function": {
        "name": "send_message",
        "arguments": {
          "message": "Last week we discussed Python decorators..."
        }
      }
    }
  ]
}
```

### Example 3: Creating a Custom Memory Block

```python
# Agent wants to track coding preferences

{
  "tool_calls": [
    {
      "function": {
        "name": "memory",
        "arguments": {
          "command": "create",
          "path": "coding_style",
          "description": "The user's coding style preferences",
          "file_text": "- Uses type hints\n- Prefers functional style\n- Likes docstrings"
        }
      }
    }
  ]
}

# BlockManager creates new block
new_block = Block(
    id="block-abc123",
    label="coding_style",
    description="The user's coding style preferences",
    value="- Uses type hints\n- Prefers functional style\n- Likes docstrings",
    limit=8000,
    created_by_id=agent.id,
)

# Block saved to database
await block_manager.create_or_update_block_async(new_block, actor)

# Now appears in system prompt
<memory_blocks>
...
<coding_style>
<description>
The user's coding style preferences
</description>
<metadata>
- chars_current=63
- chars_limit=8000
</metadata>
<value>
- Uses type hints
- Prefers functional style
- Likes docstrings
</value>
</coding_style>
</memory_blocks>
```

---

## Summary

### Memory System Diagram

```
┌──────────────────────────────────────┐
│         User Interaction             │
└────────────────┬─────────────────────┘
                 ↓
┌──────────────────────────────────────┐
│      Agent Step() Execution          │
│  - Load memory blocks from DB        │
│  - Render into system prompt         │
│  - Pass to LLM                       │
└────────────────┬─────────────────────┘
                 ↓
         ┌───────────────┐
         │  LLM Thinks   │
         └───────┬───────┘
                 ↓
    ┌────────────────────────┐
    │  Calls memory() tool?  │
    └─────┬────────────┬─────┘
          │ Yes        │ No
          ↓            ↓
┌─────────────────┐   ┌──────────────┐
│ ToolExecution   │   │ send_message │
│ Manager         │   │ or other     │
│                 │   └──────────────┘
│ 1. Parse args   │
│ 2. Validate     │
│ 3. Update block │
│ 4. Save to DB   │
│ 5. Mark dirty   │
└────────┬────────┘
         ↓
┌─────────────────────────────┐
│  _rebuild_memory_async()    │
│  - Reload blocks from DB    │
│  - Regenerate system prompt │
│  - Update system message    │
└────────┬────────────────────┘
         ↓
┌─────────────────────────────┐
│  Next LLM Call              │
│  - Sees updated memory      │
│  - Can reference new info   │
└─────────────────────────────┘
```

### Key Files Reference

| Component | File | Key Lines | Purpose |
|-----------|------|-----------|---------|
| Memory Class | `letta/schemas/memory.py` | 56-364 | Memory container |
| Block Schema | `letta/schemas/block.py` | 13-208 | Block definitions |
| Block Manager | `letta/services/block_manager.py` | 84-166 | CRUD operations |
| Passage Manager | `letta/services/passage_manager.py` | 51-100 | Archival memory |
| Memory Tool | `letta/functions/function_sets/base.py` | 9-67 | Tool interface |
| Memory Rebuild | `letta/agents/base_agent.py` | 86-193 | System prompt update |
| Summarizer | `letta/services/summarizer/summarizer.py` | - | Auto-summarization |

### Next Steps

- **Tool System**: See [tool-system.md](./tool-system.md) for tool execution details
- **Agent Execution**: See [agent-execution.md](./agent-execution.md) for how memory fits in execution loop
- **Database**: See [database-orm.md](./database-orm.md) for Block ORM details

---

**Navigation Tips:**

To view memory blocks in action:
```bash
# See how memory is rendered
cat letta-repo/letta/schemas/memory.py | sed -n '110,169p'

# See block validation
cat letta-repo/letta/schemas/block.py | sed -n '51,72p'

# See block manager operations
cat letta-repo/letta/services/block_manager.py | sed -n '139,166p'
```
