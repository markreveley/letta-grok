# Context Window

## What is a Context Window?

A **context window** is the maximum amount of text (measured in tokens) that an LLM can process at once. Think of it as the LLM's "working memory" or "attention span."

## The Fundamental Constraint

Every LLM has a **fixed context window limit**:

| Model | Context Window |
|-------|----------------|
| GPT-3.5-turbo | 16,384 tokens (~12,000 words) |
| GPT-4 | 8,192 tokens (~6,000 words) |
| GPT-4-turbo | 128,000 tokens (~96,000 words) |
| Claude 3 Sonnet | 200,000 tokens (~150,000 words) |
| Claude 3.5 Sonnet | 200,000 tokens (~150,000 words) |

**1 token ≈ 0.75 words** (English average)

---

## Why Context Windows Matter

### The Problem

```
User starts conversation:
- Message 1: "I'm working on a Python project..."
- Message 2: "It uses FastAPI and PostgreSQL..."
- Message 3: "Here's my database schema..." [large code block]
...
- Message 50: "Remember that Python project from earlier?"

If total tokens > context window limit:
❌ LLM cannot see Message 1 anymore
❌ LLM has "forgotten" the beginning of the conversation
```

### The Context Window as a Sliding Window

```
Context Window (8K tokens)
┌─────────────────────────────────────────┐
│ [System] [Msg 1] [Msg 2] ... [Msg N]   │  ← All messages fit
└─────────────────────────────────────────┘

After 100 messages:
┌─────────────────────────────────────────┐
│ [System] [Msg 85] [Msg 86] ... [Msg 100]│  ← Early messages dropped
└─────────────────────────────────────────┘
         ^ Messages 1-84 are GONE
```

---

## What Goes in the Context Window?

### Anatomy of a Letta Context Window

```
┌──────────────────────────────────────────────────┐
│ 1. System Prompt                (~1,000 tokens)  │
│    ├─ Base instructions                          │
│    ├─ Memory blocks (persona, human)             │
│    └─ Tool definitions                           │
├──────────────────────────────────────────────────┤
│ 2. Conversation History         (~5,000 tokens)  │
│    ├─ User message                               │
│    ├─ Assistant response                         │
│    ├─ Tool calls                                 │
│    └─ Tool results                               │
├──────────────────────────────────────────────────┤
│ 3. Current Input                (~500 tokens)    │
│    └─ New user message                           │
└──────────────────────────────────────────────────┘
Total: ~6,500 / 8,192 tokens (80% full)
```

### Token Distribution Example

**GPT-4 (8K context window):**
```
System Prompt:           1,200 tokens
Memory Blocks:             300 tokens
Tool Definitions:          800 tokens
─────────────────────────────────────
Overhead:                2,300 tokens
Available for messages:  5,892 tokens

With each message ~50 tokens:
→ Can fit ~118 recent messages
```

---

## The Context Window Problem

### Scenario: Long Conversation

```
Conversation starts:
Messages 1-10:    1,500 tokens total
Messages 11-50:   3,000 tokens total
Messages 51-100:  3,500 tokens total
Messages 101-150: 3,500 tokens total
─────────────────────────────────────
Total:           11,500 tokens

Context window limit: 8,000 tokens

⚠️ PROBLEM: 3,500 tokens over limit!
```

### What Happens Without Management?

**Option 1: Truncate (naive approach)**
```
Drop oldest messages until under limit
→ Agent "forgets" early conversation
→ No continuity
```

**Option 2: Fail**
```
Throw error: "Context window exceeded"
→ Conversation breaks
→ User frustrated
```

**Option 3: Summarize (Letta's approach)**
```
Compress old messages into summary
→ Preserve key information
→ Continue conversation smoothly
```

---

## How Letta Solves Context Window Limits

### The MemGPT LLM OS Approach

Letta treats the LLM context window like a computer's **RAM**:

```
Computer Memory Analogy:
┌─────────────────────────────────────────┐
│ RAM (fast, limited)                     │
│ ← Currently active data                 │
└─────────────────────────────────────────┘
         ↕ swap in/out
┌─────────────────────────────────────────┐
│ Disk (slow, unlimited)                  │
│ ← All data stored permanently           │
└─────────────────────────────────────────┘

Letta Memory:
┌─────────────────────────────────────────┐
│ Context Window (fast, limited)          │
│ ← Recent messages + core memory         │
└─────────────────────────────────────────┘
         ↕ swap in/out
┌─────────────────────────────────────────┐
│ Database (slow, unlimited)               │
│ ← Full history + archival memory        │
└─────────────────────────────────────────┘
```

### 1. Memory Hierarchy

**In-Context Memory** (in the context window):
- Recent messages (~last 50)
- Core memory blocks (editable summaries)
- Conversation summary

**Out-of-Context Memory** (in database):
- Full message history (all messages ever sent)
- Archival memory passages
- Vector embeddings for search

### 2. Automatic Summarization

When context window fills up (>90%):

```python
# Letta automatically:
1. Detects context window pressure
2. Creates a summary of old messages
3. Stores summary in a memory block
4. Removes old messages from context
5. Continues conversation with more space
```

**Example:**
```
Before Summarization (7,800 / 8,000 tokens):
├─ System prompt
├─ Memory blocks
├─ Messages 1-100 (old)
└─ Messages 101-120 (recent)

After Summarization (4,500 / 8,000 tokens):
├─ System prompt
├─ Memory blocks
│  └─ conversation_summary: "User discussed Python project,
│      database schema, deployment issues..."
├─ Messages 101-120 (recent)
└─ [3,500 tokens freed up!]
```

### 3. Selective Recall

Agent can search full history when needed:

```
User: "What did we discuss about databases last week?"

Agent:
1. Searches archival memory with "databases"
2. Retrieves relevant passages from week ago
3. Uses that info to respond
4. Never had to keep all that in context window
```

---

## Context Window Strategies

### Strategy 1: Fixed Window (Simple)

```
Keep last N messages in context
Drop older messages

Pros: Simple, predictable
Cons: Hard cutoff, loses context
```

### Strategy 2: Summarization (Letta Default)

```
When nearing limit:
1. Summarize oldest messages
2. Store summary in memory block
3. Delete original messages from context

Pros: Preserves information, continuous
Cons: Summary may lose nuance
```

### Strategy 3: Hybrid (Best)

```
Combine approaches:
1. Keep recent messages verbatim
2. Summarize older messages
3. Store important facts in memory blocks
4. Use archival search for deep history

Pros: Best of all approaches
Cons: More complex
```

---

## Monitoring Context Window Usage

### Check Current Usage

```python
# Get agent's context window status
agent = client.agents.get(agent_id)

context_overview = agent.context_window_overview
print(f"Current: {context_overview.context_window_size_current}")
print(f"Max: {context_overview.context_window_size_max}")
print(f"Usage: {context_overview.context_window_size_current / context_overview.context_window_size_max * 100}%")
```

### Warning Signs

```
⚠️ 70-80% full: Consider summarization soon
🚨 80-90% full: Summarization will trigger automatically
❌ 90%+ full: Risk of context overflow
```

---

## Best Practices

### 1. Configure Appropriate Window

```python
# Use larger models for long conversations
agent = client.agents.create(
    model="gpt-4-turbo",          # 128K context
    # vs
    model="gpt-4",                # 8K context
)
```

### 2. Use Memory Blocks Wisely

```python
# Store important persistent info in memory blocks
# Don't rely on message history for critical data
memory_blocks=[
    {
        "label": "project_context",
        "value": "Working on: E-commerce site\nTech: React, Node.js"
    }
]
```

### 3. Enable Summarization

```python
# Ensure summarization is enabled (default: true)
agent = client.agents.create(
    enable_summarization=True,
    message_buffer_limit=50,      # Summarize after 50 messages
)
```

### 4. Use Archival Memory

```python
# For reference data that doesn't need to be in context
client.agents.archival_memory.insert(
    agent_id=agent.id,
    content="Full product catalog: ..."
)

# Agent can search when needed
# Without keeping it all in context
```

---

## Common Mistakes

### ❌ Ignoring Context Limits

```python
# Sending huge messages repeatedly
for doc in large_documents:
    client.agents.messages.create(
        messages=[{"content": doc}]  # Each doc is 2K tokens
    )
# → Context fills up fast, early messages lost
```

### ✅ Better Approach

```python
# Store large docs in archival memory
for doc in large_documents:
    client.agents.archival_memory.insert(
        agent_id=agent.id,
        content=doc
    )

# Agent searches when needed
client.agents.messages.create(
    messages=[{"content": "Find info about product X"}]
)
```

---

## Summary

The **context window** is:
- The LLM's "working memory"
- Limited by model (8K to 200K+ tokens)
- A critical constraint in all LLM systems

Letta solves context window limits through:
- ✅ Memory hierarchy (in-context + out-of-context)
- ✅ Automatic summarization
- ✅ Searchable archival memory
- ✅ Editable memory blocks

**Key Insight**: Letta lets you have infinite conversations by managing what's in the limited context window intelligently.

**Related Concepts:**
- [Memory Hierarchy](./03-memory-hierarchy.md) - How Letta organizes memory
- [Summarization](./11-summarization.md) - Automatic compression
- [Archival Memory](./05-archival-memory.md) - Unlimited storage
