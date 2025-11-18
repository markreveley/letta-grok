# Memory Hierarchy

## What is Memory Hierarchy?

**Memory hierarchy** is Letta's solution to the context window problem. It organizes agent memory into multiple tiers, similar to how computer memory works (RAM, disk, cache).

## The Core Idea: LLM Operating System

Letta is based on the **MemGPT** research paper, which introduced the concept of an "LLM Operating System" for memory management.

### Computer Memory Analogy

```
Traditional Computer:
┌──────────────────────────────────────┐
│ CPU Registers (fastest, tiny)        │
├──────────────────────────────────────┤
│ L1/L2 Cache (very fast, small)       │
├──────────────────────────────────────┤
│ RAM (fast, limited ~16GB)            │
├──────────────────────────────────────┤
│ SSD/Disk (slow, large ~1TB)          │
└──────────────────────────────────────┘

OS swaps data between layers as needed
```

```
Letta Memory Hierarchy:
┌──────────────────────────────────────┐
│ Core Memory Blocks (editable)        │
├──────────────────────────────────────┤
│ Recent Messages (last N)             │
├──────────────────────────────────────┤
│ Context Window (fast, ~8K tokens)    │
├──────────────────────────────────────┤
│ Full Message History (database)      │
├──────────────────────────────────────┤
│ Archival Memory (searchable, ∞)      │
└──────────────────────────────────────┘

Agent swaps data between layers using tools
```

---

## The Three Tiers

### Tier 1: In-Context Memory (Fast Access)

**What's included:**
- System prompt
- Core memory blocks (persona, human, custom)
- Recent messages
- Conversation summary

**Characteristics:**
- ✅ Immediate access (LLM sees it directly)
- ✅ Fast - no search needed
- ❌ Limited size (context window constraint)
- ❌ Volatile - old messages evicted

**Example:**
```
Context Window Content:
┌──────────────────────────────────────────────┐
│ SYSTEM PROMPT                                │
│ You are a helpful assistant...               │
│                                              │
│ <memory_blocks>                              │
│   <human>Name: Sarah, Engineer</human>      │
│   <persona>I am a coding tutor</persona>    │
│ </memory_blocks>                             │
│                                              │
│ RECENT MESSAGES:                             │
│ User: "Explain recursion"                    │
│ Assistant: "Recursion is when..."           │
│ User: "Show me an example"                   │
│ Assistant: [current response]                │
└──────────────────────────────────────────────┘
```

### Tier 2: Recall Memory (Full History)

**What's included:**
- Every message ever sent
- Complete conversation log
- Metadata (timestamps, tokens, etc.)

**Characteristics:**
- ✅ Complete history
- ✅ Searchable
- ❌ Not immediately accessible (must search)
- ❌ Requires database query

**Example:**
```sql
SELECT * FROM messages 
WHERE agent_id = 'agent-123'
ORDER BY created_at DESC
LIMIT 1000;

Returns: All 1,000 messages from this agent
```

### Tier 3: Archival Memory (Long-term Knowledge)

**What's included:**
- Important facts and information
- Summarized old conversations
- Reference documents
- Knowledge base entries

**Characteristics:**
- ✅ Unlimited storage
- ✅ Semantic search via embeddings
- ❌ Requires explicit search
- ❌ Slower than in-context

**Example:**
```python
# Agent searches archival memory
results = search_archival_memory(
    query="Python project details",
    top_k=5
)

# Returns most relevant passages:
[
    "User is building Python web app with FastAPI",
    "Database schema uses PostgreSQL with...",
    "Deployment on AWS using Docker..."
]
```

---

## How Data Moves Between Tiers

### Upward Movement (Disk → RAM)

**1. Search and Retrieve**

```python
User: "What did we discuss about databases?"

Agent Flow:
1. Searches archival memory for "databases"
2. Retrieves top 3 relevant passages
3. Temporarily adds to context window
4. Responds using retrieved information
5. Passages NOT permanently in context
```

**2. Recall Recent Messages**

```python
Agent starting new session:
1. Loads last 50 messages from database
2. Puts them in context window
3. Rebuilds full conversation state
```

### Downward Movement (RAM → Disk)

**1. Message Persistence**

```python
Every message exchange:
1. User sends message → saved to database
2. Agent responds → saved to database
3. Both stay in context window (for now)
```

**2. Eviction (Context Window Full)**

```python
When context nears limit:
1. Oldest messages removed from context
2. Remain in database (recall memory)
3. Optionally: Summarized to archival memory
```

**3. Explicit Archival**

```python
Agent tool call:
archival_memory_insert(
    content="Important: User prefers tabs over spaces"
)

Result:
- Stored permanently in archival memory
- Can be searched semantically
- Survives context eviction
```

---

## Memory Hierarchy in Action

### Example Conversation Timeline

```
Session 1 - Day 1:
├─ Messages 1-20 created
├─ All in context window
└─ All saved to database

Session 2 - Day 2:
├─ Load last 50 messages to context
├─ Messages 21-40 created
├─ Context nearing limit (70 messages total)
└─ All saved to database

Session 3 - Day 3:
├─ Load last 50 messages to context
├─ Context at limit (80 messages total)
├─ SUMMARIZATION TRIGGERED
│   ├─ Messages 1-40 summarized
│   ├─ Summary saved to archival memory
│   └─ Messages 1-40 removed from context
├─ Context now has: Messages 41-80
└─ Continue conversation

Session 4 - Week 2:
├─ User asks about early conversation
├─ Agent searches archival memory
├─ Finds summary of Messages 1-40
├─ Responds with context from summary
└─ Never needed original messages in context
```

---

## Managing Memory Tiers

### Core Memory Blocks (Tier 1)

**Purpose**: Store critical, frequently-accessed information

```python
# User's key information
human_block = {
    "label": "human",
    "value": """
    Name: Sarah Chen
    Role: Senior Engineer
    Interests: Python, distributed systems
    Learning: Kubernetes
    """
}

# Agent's identity
persona_block = {
    "label": "persona", 
    "value": """
    I am a technical mentor focused on backend development.
    I provide detailed explanations with code examples.
    """
}
```

**Best Practices:**
- ✅ Keep concise (each block has character limit)
- ✅ Update as user provides new information
- ✅ Store only essential facts

### Recall Memory (Tier 2)

**Purpose**: Complete conversation history

```python
# Search recent messages
messages = client.agents.messages.list(
    agent_id="agent-123",
    limit=100
)

# Full message retrieval when needed
# Mostly automatic - Letta manages this
```

**Best Practices:**
- ✅ Let Letta manage automatically
- ✅ Trust that history is preserved
- ❌ Don't try to manually manage

### Archival Memory (Tier 3)

**Purpose**: Long-term knowledge base

```python
# Agent inserts important information
client.agents.archival_memory.insert(
    agent_id="agent-123",
    content="User's project architecture: Microservices with..."
)

# Agent searches when relevant
results = client.agents.archival_memory.search(
    agent_id="agent-123",
    query="project architecture",
    top_k=3
)
```

**Best Practices:**
- ✅ Use for reference information
- ✅ Add important discoveries
- ✅ Tag passages for better search
- ❌ Don't duplicate core memory

---

## When to Use Each Tier

### Use In-Context Memory (Core Blocks) For:
- User's name and basic info
- Agent's personality and role
- Current project/task context
- Frequently-referenced facts

### Use Recall Memory For:
- Recent conversation flow
- Context from last few exchanges
- Sequential information

### Use Archival Memory For:
- Long-term knowledge
- Reference documentation
- Summarized old conversations
- Project history and decisions

---

## Comparison Table

| Feature | In-Context | Recall | Archival |
|---------|-----------|--------|----------|
| **Access Speed** | Instant | Fast (DB query) | Medium (vector search) |
| **Storage Limit** | Context window | Unlimited | Unlimited |
| **Search Method** | N/A (always present) | SQL query | Semantic similarity |
| **Best For** | Current facts | Recent history | Long-term knowledge |
| **Agent Access** | Automatic | Manual search | Manual search |
| **Persistence** | Session-based | Permanent | Permanent |

---

## Advanced: Memory Swapping

Letta agents actively manage what's in the context window:

```python
def intelligent_memory_management():
    """
    Agent's internal process (conceptual):
    """
    
    # Check context window usage
    if context_usage > 0.8:
        # Getting full, need to make space
        
        # Option 1: Summarize old messages
        old_messages = messages[:30]
        summary = create_summary(old_messages)
        archival_memory_insert(summary)
        remove_from_context(old_messages)
        
        # Option 2: Move to archival
        important_info = extract_facts(old_messages)
        archival_memory_insert(important_info)
        
        # Option 3: Update core memory
        user_prefs = extract_preferences(old_messages)
        core_memory_replace(
            label="human",
            new_content=user_prefs
        )
    
    # Search archival if needed
    if user_query_needs_history():
        relevant_passages = archival_memory_search(
            query=extract_query()
        )
        use_in_response(relevant_passages)
```

---

## Summary

**Memory Hierarchy** enables Letta agents to have:
- ✅ Unlimited conversation history
- ✅ Fast access to recent context
- ✅ Semantic search of old information
- ✅ Efficient use of limited context window

**Three Tiers:**
1. **In-Context** (Tier 1): Fast, limited, always accessible
2. **Recall** (Tier 2): Complete history, searchable
3. **Archival** (Tier 3): Long-term knowledge, semantic search

This hierarchy is what makes Letta agents truly "stateful" and capable of long-term relationships.

**Related Concepts:**
- [Context Window](./02-context-window.md) - The constraint that necessitates hierarchy
- [Memory Blocks](./04-memory-blocks.md) - In-context editable memory
- [Archival Memory](./05-archival-memory.md) - Long-term storage
- [Summarization](./11-summarization.md) - Moving data between tiers
