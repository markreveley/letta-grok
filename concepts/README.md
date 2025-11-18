# Letta Concepts

This folder contains concept-focused explanations of the fundamental ideas behind Letta. Each document explains **what** a concept is, **why** it matters, and **how** it works in Letta.

## Core Concepts Index

### Foundational Concepts

1. [**Stateful Agents**](./01-stateful-agents.md)
   - What makes an agent "stateful"
   - Difference from stateless chatbots
   - Persistence and continuity

2. [**Context Window**](./02-context-window.md)
   - Understanding LLM token limits
   - Why context windows matter
   - Managing limited context

3. [**Memory Hierarchy**](./03-memory-hierarchy.md)
   - In-context vs out-of-context memory
   - The "LLM Operating System" concept
   - RAM vs Disk analogy

### Memory Concepts

4. [**Memory Blocks**](./04-memory-blocks.md)
   - Editable sections of memory
   - Core memory blocks (persona, human)
   - Self-editing memory

5. [**Archival Memory**](./05-archival-memory.md)
   - Long-term memory storage
   - Semantic search with embeddings
   - When to use archival memory

6. [**Embeddings & Vector Search**](./06-embeddings-vector-search.md)
   - What are embeddings
   - Semantic similarity
   - Vector databases

### Agent Behavior

7. [**Tool Calling (Function Calling)**](./07-tool-calling.md)
   - How agents use tools
   - Tool schemas and parameters
   - Built-in vs custom tools

8. [**Agent Lifecycle**](./08-agent-lifecycle.md)
   - Creating an agent
   - Agent execution loop
   - Persistence and state management

9. [**System Prompts**](./09-system-prompts.md)
   - Instructions that guide behavior
   - Dynamic system prompt generation
   - Prompt engineering for agents

### Advanced Concepts

10. [**Message History**](./10-message-history.md)
    - Conversation persistence
    - Message roles and types
    - Recall memory

11. [**Summarization**](./11-summarization.md)
    - Automatic conversation compression
    - Context window pressure
    - Summary blocks

12. [**Multi-Agent Systems**](./12-multi-agent-systems.md)
    - Agent-to-agent communication
    - Shared memory blocks
    - Orchestration patterns

13. [**Streaming**](./13-streaming.md)
    - Real-time responses
    - Token-by-token streaming
    - Server-Sent Events (SSE)

### Technical Concepts

14. [**LLM Providers**](./14-llm-providers.md)
    - Different AI model backends
    - OpenAI, Anthropic, local models
    - Provider abstraction

15. [**Sandboxing**](./15-sandboxing.md)
    - Safe code execution
    - Isolation environments
    - Security considerations

16. [**Personas & Humans**](./16-personas-humans.md)
    - Agent identity (persona)
    - User modeling (human)
    - Role-based memory

---

## How to Use This Section

### If you're new to Letta:
1. Start with **Stateful Agents** to understand the core idea
2. Read **Context Window** to understand the fundamental constraint
3. Continue with **Memory Hierarchy** to see how Letta solves it

### If you're building with Letta:
- Focus on **Memory Blocks**, **Tool Calling**, and **System Prompts**
- Understand **Archival Memory** for long-term storage
- Learn **Streaming** for real-time UIs

### If you're contributing:
- Read all foundational and memory concepts
- Understand **Agent Lifecycle** deeply
- Study **Multi-Agent Systems** for advanced features

---

## Concept vs Implementation

**Concepts** (this folder) explain the **ideas** behind Letta.

**Analysis** (../analysis/) explains the **implementation** with code references.

**Example:**
- **Concept**: [Memory Blocks](./04-memory-blocks.md) - "What are memory blocks and why do we need them?"
- **Implementation**: [Memory Management Analysis](../analysis/memory-management.md) - "How are memory blocks implemented in code?"

---

## Visual Learning

Many concepts include:
- 📊 Diagrams showing how things work
- 💡 Examples with before/after scenarios
- ⚠️ Common pitfalls and misconceptions
- 🔗 Links to relevant code and documentation

---

**See also:**
- [Main README](../README.md) - Repository overview
- [Architecture Analysis](../analysis/ARCHITECTURE_ANALYSIS.md) - System architecture
- [Quick Reference](../analysis/QUICK_REFERENCE.md) - Developer guide
