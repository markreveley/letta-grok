# Stateful Agents

## What is a Stateful Agent?

A **stateful agent** is an AI agent that maintains and manages its own persistent state across conversations. Unlike traditional chatbots that treat each interaction independently, stateful agents remember past interactions, learn from them, and evolve over time.

## The Problem with Stateless Chatbots

### Traditional Chatbot (Stateless)
```
User: "My name is Sarah"
Bot: "Nice to meet you, Sarah!"

[5 minutes later, new conversation]

User: "What's my name?"
Bot: "I don't have information about your name."
```

**Why?** The chatbot has no memory between conversations. Each interaction starts fresh.

### Stateful Agent (Letta)
```
User: "My name is Sarah"
Agent: "Nice to meet you, Sarah!"
        [Internally: Updates memory block with "Name: Sarah"]

[5 minutes later, SAME agent]

User: "What's my name?"
Agent: "Your name is Sarah."
```

**Why?** The agent stores information in persistent memory that survives across conversations.

---

## Key Characteristics of Stateful Agents

### 1. Persistent Memory

The agent's memory is stored in a database and survives:
- Application restarts
- Server reboots
- Days, weeks, months between conversations

**Example:**
```python
# Day 1
client.agents.messages.create(
    agent_id="agent-123",
    messages=[{"role": "user", "content": "I love Python"}]
)
# Agent updates its memory: "User prefers Python"

# Day 30 (same agent)
client.agents.messages.create(
    agent_id="agent-123",
    messages=[{"role": "user", "content": "Recommend a language"}]
)
# Agent: "Based on your preference for Python, I recommend..."
```

### 2. Self-Editing Memory

Agents can **modify their own memory** using tools:

```
Agent thinks: "The user just corrected their name"
Agent action: Call memory() tool to update human block
Result: Memory now reflects correct information
```

This is different from systems where only developers can modify agent memory.

### 3. Continuous Learning

As agents interact, they:
- Accumulate conversation history
- Update their understanding of the user
- Refine their persona and behavior
- Build a knowledge base over time

---

## Stateful vs Stateless: The Analogy

### Stateless (Traditional)
```
Like talking to someone with amnesia:
- Polite and helpful
- But no memory of previous conversations
- You must re-explain context every time
```

### Stateful (Letta)
```
Like talking to a long-time colleague:
- Remembers your preferences
- Knows your history together
- Builds on past interactions
- Evolves the relationship over time
```

---

## How Letta Implements Statefulness

### 1. Unique Agent Identity

Each agent has a unique ID that persists:

```python
agent = client.agents.create(
    name="My Assistant",
    ...
)
# agent.id = "agent-d9be0846"

# This ID is permanent - use it to continue conversations
```

### 2. Database-Backed State

All agent state is stored in a database:

```
Database Tables:
├── agents         # Agent configuration
├── messages       # Full conversation history
├── blocks         # Memory blocks (editable)
└── passages       # Archival memory (searchable)
```

### 3. State Snapshots

At any point, you can export an agent's complete state:

```python
# Export agent state (including all memory)
agent_file = client.agents.export_agent_serialized(agent_id)

# Import on another server
restored_agent = client.agents.import_agent_serialized(agent_file)

# Agent continues exactly where it left off
```

---

## Benefits of Stateful Agents

### 1. Continuity

```
Conversation 1:
User: "I'm working on a React project"
Agent: "Great! What are you building?"

Conversation 2 (days later):
User: "I'm stuck on a component"
Agent: "In your React project? What's the issue?"
```

The agent remembers the context without you re-explaining.

### 2. Personalization

```
Over time, the agent learns:
- Your coding style preferences
- Your level of expertise
- Your communication preferences
- Your past projects and interests

Responses become increasingly tailored to YOU.
```

### 3. Relationship Building

```
Week 1: Formal, generic responses
Week 4: Uses your name, knows your preferences
Week 12: Feels like a trusted colleague
```

---

## State Components in Letta

A stateful agent maintains multiple types of state:

### 1. **Configuration State** (rarely changes)
- System prompt
- LLM model and settings
- Available tools
- Context window size

### 2. **Memory State** (agent-editable)
- Memory blocks (persona, human, custom)
- Archival memory passages
- Conversation summary

### 3. **Conversation State** (grows over time)
- Full message history
- Tool call history
- Usage statistics

### 4. **Execution State** (transient)
- Current run ID
- In-progress steps
- Active context window

---

## Common Misconceptions

### ❌ "The agent remembers everything in the conversation"

**Reality**: The agent has a limited context window. Older messages are summarized or moved to archival memory.

### ❌ "Stateful = slower"

**Reality**: State is loaded efficiently from database. Most operations are just as fast as stateless systems.

### ❌ "I need a new agent for each user"

**Reality**: While you CAN do this, you can also use a single agent with user-specific memory blocks.

---

## Example: Building a Stateful Learning Assistant

### Initial Creation
```python
agent = client.agents.create(
    model="gpt-4",
    memory_blocks=[
        {
            "label": "student",
            "value": "Name: Unknown\nLevel: Unknown\nInterests: Unknown"
        },
        {
            "label": "persona",
            "value": "I am a patient tutor who adapts to the student's level"
        }
    ]
)
```

### First Interaction
```
User: "Hi, I'm Alex and I'm new to programming"
Agent: [Updates memory: "Name: Alex, Level: Beginner"]
Agent: "Great to meet you, Alex! Let's start with the basics..."
```

### Week 2
```
User: "I understood variables, what's next?"
Agent: [Has memory of Alex being a beginner who learned variables]
Agent: "Perfect! Let's move on to functions. Remember how variables store values?..."
```

### Month 2
```
User: "Explain recursion"
Agent: [Has memory of Alex's progression through concepts]
Agent: "Given your understanding of functions and loops, recursion is..."
```

---

## When to Use Stateful Agents

### ✅ Good Use Cases

1. **Personal Assistants** - Need to remember user preferences and history
2. **Customer Support** - Track ongoing issues and user history
3. **Tutoring/Coaching** - Remember student progress and learning style
4. **Long-term Projects** - Maintain context across weeks/months
5. **Relationship-based Interactions** - Value continuity and personalization

### ⚠️ Consider Alternatives

1. **One-off Queries** - If no continuity needed, stateless is simpler
2. **Strict Privacy Requirements** - If you can't store any conversation data
3. **Completely Independent Requests** - No benefit from state

---

## Code References

**Where to learn more about implementation:**

- Agent State Schema: [`letta/schemas/agent.py`](../letta-repo/letta/schemas/agent.py)
- Agent Manager: [`letta/services/agent_manager.py`](../letta-repo/letta/services/agent_manager.py)
- Agent ORM: [`letta/orm/agent.py`](../letta-repo/letta/orm/agent.py)

**Related Concepts:**
- [Memory Hierarchy](./03-memory-hierarchy.md) - How state is organized
- [Memory Blocks](./04-memory-blocks.md) - Editable state
- [Agent Lifecycle](./08-agent-lifecycle.md) - State management details

---

## Summary

**Stateful Agents** are the foundation of Letta. They enable:
- ✅ Persistent memory across conversations
- ✅ Self-editing memory
- ✅ Continuous learning and adaptation
- ✅ Personalized, context-aware interactions

This transforms AI agents from simple chatbots into long-term collaborators that grow more useful over time.
