# Memory Blocks

## What are Memory Blocks?

**Memory blocks** are editable sections of an agent's in-context memory. Think of them as "variables" or "note cards" that the agent can read and modify during conversations.

## The Core Concept

```
Traditional Chatbot:
System Prompt: "You are a helpful assistant"
[Fixed - cannot be changed]

Letta Agent:
System Prompt: "You are a helpful assistant"
Memory Blocks:
  <human>Name: Unknown</human>
  <persona>I am a helpful assistant</persona>
[Editable - agent can update these!]
```

---

## Why Memory Blocks?

### The Problem

```
User: "My name is John"
Agent: "Nice to meet you, John"

[50 messages later, message with "My name is John" falls out of context window]

User: "What's my name?"
Agent: "I don't have that information"
```

### The Solution

```
User: "My name is John"
Agent internal process:
  1. "I should remember this"
  2. Calls memory() tool
  3. Updates human block: "Name: John"

[Block stays in context window permanently]

User: "What's my name?"
Agent: "Your name is John" [reads from human block]
```

---

## Default Memory Blocks

### 1. Human Block

Stores information about the user.

```xml
<human>
<description>
Information about the human you are conversing with.
</description>
<metadata>
- chars_current=85
- chars_limit=8000
</metadata>
<value>
Name: Sarah Chen
Occupation: Software Engineer
Interests: Python, distributed systems
Current project: Building microservices platform
</value>
</human>
```

**Purpose:**
- User's name and role
- Preferences and interests
- Current context
- Important facts to remember

### 2. Persona Block

Defines the agent's identity and behavior.

```xml
<persona>
<description>
Your persona and role in this conversation.
</description>
<metadata>
- chars_current=120
- chars_limit=8000
</metadata>
<value>
I am a technical mentor specializing in backend development.
I provide detailed explanations with code examples.
I adapt my responses to the user's experience level.
I encourage best practices and clean code.
</value>
</persona>
```

**Purpose:**
- Agent's role and expertise
- Communication style
- Goals and objectives
- Behavioral guidelines

---

## How Agents Edit Memory Blocks

### The memory() Tool

Agents can modify blocks using the `memory()` tool:

```python
# Agent internal reasoning (conceptual):
"The user just told me they prefer tabs over spaces.
I should update my memory."

# Agent action:
memory(
    command="str_replace",
    path="human",
    old_str="Code style: Unknown",
    new_str="Code style: Prefers tabs, functional programming"
)
```

### Real Example

```
Conversation:
User: "I prefer functional programming and tabs"
Agent: "Got it, I'll remember that!"

Behind the scenes:
┌──────────────────────────────────────┐
│ Agent calls memory() tool:           │
├──────────────────────────────────────┤
│ {                                    │
│   "command": "str_replace",          │
│   "path": "human",                   │
│   "old_str": "Programming style: ?", │
│   "new_str": "Prefers functional     │
│                programming, uses     │
│                tabs not spaces"      │
│ }                                    │
└──────────────────────────────────────┘

Memory block updated in database ✓
System prompt rebuilt with new memory ✓

Next message sees updated memory ✓
```

---

## Creating Custom Memory Blocks

Beyond `human` and `persona`, you can create custom blocks:

### Example: Project Context Block

```python
agent = client.agents.create(
    memory_blocks=[
        {"label": "human", "value": "Name: Alex"},
        {"label": "persona", "value": "I am a coding tutor"},
        {"label": "current_project", "value": """
        Project: E-commerce Platform
        Tech Stack: React, Node.js, PostgreSQL
        Stage: MVP development
        Next Milestone: User authentication
        """}
    ]
)
```

Now the agent always has project context in its memory!

### Example: Learning Progress Block

```python
{
    "label": "learning_progress",
    "value": """
    Completed Topics:
    ✓ Variables and data types
    ✓ Functions and scope
    ✓ Classes and objects
    
    Currently Learning:
    - Async/await patterns
    
    Struggling With:
    - Recursion
    
    Next Up:
    - Error handling
    """
}
```

---

## Memory Block Properties

### Structure

```python
class Block:
    id: str                    # Unique identifier
    label: str                 # Name (e.g., "human", "persona")
    value: str                 # Content (the actual memory)
    limit: int = 8000         # Character limit
    description: str           # What this block is for
    read_only: bool = False   # Can agent edit it?
```

### Character Limits

**Default: 8,000 characters per block**

```
Why limits?
1. Prevent context window overflow
2. Force concise, relevant information
3. Encourage good information architecture
```

**Example of hitting limit:**
```python
# Agent tries to update
memory(
    path="human",
    new_str="..." + 9000_chars + "..."
)

# Result:
❌ Error: "Edit failed: Exceeds 8000 character limit (requested 9000)"
```

**Solution: Multiple blocks or archival memory**
```python
# Option 1: Split into multiple blocks
blocks = [
    {"label": "user_profile", "value": "..."},
    {"label": "user_preferences", "value": "..."},
    {"label": "user_history", "value": "..."}
]

# Option 2: Use archival memory for large data
archival_memory_insert(
    content="Full user history: ..."
)
```

---

## Memory Block Operations

### Read

Agents automatically see all blocks in system prompt.

```xml
System message to LLM:
<memory_blocks>
  <human>Name: John</human>
  <persona>I am helpful</persona>
  <project>Building app</project>
</memory_blocks>
```

### Update (Replace Text)

```python
memory(
    command="str_replace",
    path="human",
    old_str="Name: Unknown",
    new_str="Name: John Smith"
)
```

### Update (Insert at Line)

With line-numbered blocks (Anthropic models):

```python
memory(
    command="insert",
    path="human",
    insert_line=3,
    insert_text="Email: john@example.com"
)
```

### Create New Block

```python
memory(
    command="create",
    path="preferences",
    description="User's preferences and settings",
    file_text="Theme: Dark mode\nNotifications: Enabled"
)
```

### Delete Block

```python
memory(
    command="delete",
    path="old_project"
)
```

---

## Best Practices

### ✅ DO: Keep Blocks Focused

```python
# Good - each block has clear purpose
{
    "label": "user_profile",
    "value": "Name: Sarah\nRole: Engineer"
},
{
    "label": "current_task",
    "value": "Building authentication system"
}

# Bad - everything in one block
{
    "label": "human",
    "value": "Name: Sarah, Engineer, building auth, likes Python, has cat..."
}
```

### ✅ DO: Update Regularly

```python
# Agent should update as it learns
User: "I switched to TypeScript"
Agent: [Updates preferences block to reflect change]
```

### ✅ DO: Use Descriptive Labels

```python
# Good
"label": "learning_progress"
"label": "project_requirements"
"label": "code_style_preferences"

# Bad
"label": "misc"
"label": "stuff"
"label": "block1"
```

### ❌ DON'T: Exceed Character Limits

```python
# Will fail
value = "x" * 10000  # Over 8000 limit

# Better: Split or summarize
value = "Summary: ..." # Under limit
```

### ❌ DON'T: Duplicate Information

```python
# Bad - same info in multiple blocks
human_block: "Project: React app"
project_block: "Type: React app"

# Good - info in one place
project_block: "Type: React app, Framework: Next.js"
```

---

## Advanced: Read-Only Blocks

Prevent agent from editing certain blocks:

```python
{
    "label": "system_rules",
    "value": "1. Always be respectful\n2. Cite sources",
    "read_only": True  # Agent cannot modify this
}
```

Use cases:
- Company policies
- Safety guidelines
- Fixed instructions

---

## Memory Blocks vs Other Memory

| Feature | Memory Blocks | Message History | Archival Memory |
|---------|---------------|-----------------|-----------------|
| **Location** | Always in context | Recent only | Database |
| **Size** | Limited (8K chars) | Limited by context | Unlimited |
| **Editable** | Yes (by agent) | No | No |
| **Purpose** | Key facts | Conversation flow | Long-term knowledge |
| **Access** | Immediate | Immediate (recent) | Search required |

---

## Real-World Example

### Personal Assistant Agent

```python
agent = client.agents.create(
    memory_blocks=[
        {
            "label": "human",
            "value": """
            Name: Alex Rodriguez
            Occupation: Product Manager at TechCorp
            Location: San Francisco, PT timezone
            Work schedule: 9-5 PT, Meetings Tue/Thu mornings
            """
        },
        {
            "label": "persona",
            "value": """
            I am Alex's executive assistant.
            I help with scheduling, reminders, and task management.
            I'm proactive about conflicts and time management.
            I know Alex's preferences and anticipate needs.
            """
        },
        {
            "label": "ongoing_tasks",
            "value": """
            - Q4 Planning presentation (due Friday)
            - Interview candidate Sarah (scheduled Tuesday 2pm)
            - Review product roadmap (in progress)
            - Team offsite planning (pending)
            """
        },
        {
            "label": "preferences",
            "value": """
            - Prefers morning meetings
            - Needs 30min prep time before important calls
            - Likes concise summaries
            - Wants end-of-day task review
            """
        }
    ]
)
```

### Agent Updates Memory During Conversation

```
User: "The Q4 presentation is done"
Agent: "Great! I'll update your task list."
       [Removes Q4 presentation from ongoing_tasks block]

User: "Schedule a meeting with Sarah for next week"
Agent: "I see you're already interviewing Sarah on Tuesday.
        Should this be a follow-up?"
       [Read from ongoing_tasks block]
```

---

## Summary

**Memory Blocks** are the agent's "working memory":
- ✅ Always in context window
- ✅ Editable by agent
- ✅ Persistent across conversations
- ✅ Structured and organized

**Key Benefits:**
- Information survives context window eviction
- Agent actively maintains its own memory
- Organized structure vs unstructured chat history
- Fast access (no search needed)

**Default blocks:** `human` (user info) and `persona` (agent identity)

**Custom blocks:** Create for any persistent structured data

**Code References:**
- Block Schema: [`letta/schemas/block.py`](../letta-repo/letta/schemas/block.py)
- Block Manager: [`letta/services/block_manager.py`](../letta-repo/letta/services/block_manager.py)
- Memory Tool: [`letta/functions/function_sets/base.py`](../letta-repo/letta/functions/function_sets/base.py)

**Related Concepts:**
- [Memory Hierarchy](./03-memory-hierarchy.md) - Where blocks fit
- [Tool Calling](./07-tool-calling.md) - How agents edit blocks
- [System Prompts](./09-system-prompts.md) - How blocks are rendered
