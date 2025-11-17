# Agent Execution System - Deep Dive

This document provides a detailed analysis of how agents execute in Letta, with specific code references and line numbers for navigation.

## Table of Contents
- [Overview](#overview)
- [Agent Class Hierarchy](#agent-class-hierarchy)
- [Core Execution Flow](#core-execution-flow)
- [Step-by-Step Breakdown](#step-by-step-breakdown)
- [Tool Execution](#tool-execution)
- [Memory Updates](#memory-updates)
- [Streaming vs Non-Streaming](#streaming-vs-non-streaming)
- [Error Handling](#error-handling)

---

## Overview

The agent execution system in Letta is responsible for processing user messages, managing context, calling LLMs, executing tools, and maintaining memory. The main execution happens through the `step()` and `step_stream()` methods.

**Primary Files:**
- [`letta-repo/letta/agents/base_agent.py`](../letta-repo/letta/agents/base_agent.py) - Abstract base class (202 lines)
- [`letta-repo/letta/agents/letta_agent.py`](../letta-repo/letta/agents/letta_agent.py) - Main agent implementation (1,904 lines)
- [`letta-repo/letta/agents/letta_agent_v2.py`](../letta-repo/letta/agents/letta_agent_v2.py) - Enhanced version
- [`letta-repo/letta/agents/letta_agent_v3.py`](../letta-repo/letta/agents/letta_agent_v3.py) - Latest iteration

---

## Agent Class Hierarchy

### BaseAgent (Abstract)

**Location:** [`letta/agents/base_agent.py:28-202`](../letta-repo/letta/agents/base_agent.py#L28-L202)

```python
class BaseAgent(ABC):
    """
    Abstract base class for AI agents, handling message management,
    tool execution, and context tracking.
    """

    def __init__(
        self,
        agent_id: str,
        openai_client: Optional[openai.AsyncClient],  # Line 38
        message_manager: MessageManager,               # Line 39
        agent_manager: AgentManager,                   # Line 40
        actor: User,                                   # Line 41
    ):
        self.agent_id = agent_id                       # Line 43
        self.openai_client = openai_client             # Line 44
        self.message_manager = message_manager         # Line 45
        self.agent_manager = agent_manager             # Line 46
        self.passage_manager = PassageManager()        # Line 48
        self.actor = actor                             # Line 49
        self.logger = get_logger(agent_id)             # Line 50
```

**Key Methods:**
- `step()` - Abstract method at line 53
- `step_stream()` - Abstract method at line 62
- `_rebuild_memory_async()` - Memory reconstruction at line 86-193
- `pre_process_input_message()` - Input preprocessing at line 71-84

### LettaAgent (Concrete Implementation)

**Location:** [`letta/agents/letta_agent.py:77-1904`](../letta-repo/letta/agents/letta_agent.py#L77-L1904)

**Constructor:** Lines 78-156

```python
class LettaAgent(BaseAgent):
    def __init__(
        self,
        agent_id: str,                                 # Line 80
        message_manager: MessageManager,               # Line 81
        agent_manager: AgentManager,                   # Line 82
        block_manager: BlockManager,                   # Line 83
        job_manager: JobManager,                       # Line 84
        passage_manager: PassageManager,               # Line 85
        actor: User,                                   # Line 86
        step_manager: StepManager = NoopStepManager(), # Line 87
        telemetry_manager: TelemetryManager = ...,     # Line 88
        current_run_id: str | None = None,             # Line 89

        ## Summarizer settings
        summarizer_mode: SummarizationMode = ...,      # Line 91
        summary_block_label: str = ...,                # Line 93
        message_buffer_limit: int = ...,               # Line 94
        enable_summarization: bool = ...,              # Line 96
        # ... more config
    ):
```

**Key Attributes:**
- `response_messages: List[Message] = []` - Accumulates messages during execution
- `summarizer: Summarizer` - Handles memory summarization
- `tool_execution_manager: ToolExecutionManager` - Executes tool calls

---

## Core Execution Flow

### Entry Point: `step()` Method

**Location:** [`letta/agents/letta_agent.py:169-205`](../letta-repo/letta/agents/letta_agent.py#L169-L205)

```python
async def step(
    self,
    input_messages: list[MessageCreateBase],           # Line 171
    max_steps: int = DEFAULT_MAX_STEPS,                # Line 172
    run_id: str | None = None,                         # Line 173
    use_assistant_message: bool = True,                # Line 174
    request_start_timestamp_ns: int | None = None,     # Line 175
    include_return_message_types: list[MessageType] | None = None,  # Line 176
    dry_run: bool = False,                             # Line 177
) -> Union[LettaResponse, dict]:
    """
    Main entry point for agent execution.

    Flow:
    1. Load agent state from database (line 180-184)
    2. Execute internal _step() method (line 185-192)
    3. Return LettaResponse or dry_run payload (line 194-205)
    """

    # [DB Call] Load agent with all relationships
    agent_state = await self.agent_manager.get_agent_by_id_async(
        agent_id=self.agent_id,                        # Line 181
        include_relationships=[                        # Line 182
            "tools",                                   # Tool definitions
            "memory",                                  # Memory blocks
            "tool_exec_environment_variables",         # Environment vars
            "sources"                                  # Attached data sources
        ],
        actor=self.actor,                              # Line 183
    )

    # Execute the main logic
    result = await self._step(                         # Line 185
        agent_state=agent_state,
        input_messages=input_messages,
        max_steps=max_steps,
        run_id=run_id,
        request_start_timestamp_ns=request_start_timestamp_ns,
        dry_run=dry_run,
    )

    # Return formatted response
    if dry_run:
        return result  # Return request payload for inspection

    _, new_in_context_messages, stop_reason, usage = result  # Line 198
    return _create_letta_response(                     # Line 199-205
        new_in_context_messages=new_in_context_messages,
        use_assistant_message=use_assistant_message,
        stop_reason=stop_reason,
        usage=usage,
        include_return_message_types=include_return_message_types,
    )
```

### Internal: `_step()` Method

**Location:** [`letta/agents/letta_agent.py:551-862`](../letta-repo/letta/agents/letta_agent.py#L551-L862)

This is the core execution logic (312 lines). Here's the breakdown:

```python
@trace_method  # OpenTelemetry tracing
async def _step(
    self,
    agent_state: AgentState,                           # Line 553
    input_messages: list[MessageCreateBase],           # Line 554
    max_steps: int = DEFAULT_MAX_STEPS,                # Line 555
    run_id: str | None = None,                         # Line 556
    request_start_timestamp_ns: int | None = None,     # Line 557
    dry_run: bool = False,                             # Line 558
) -> tuple:
    """
    Internal execution method.

    Returns: (agent_state, new_in_context_messages, stop_reason, usage)
    """
```

**Phase 1: Initialization (Lines 560-600)**

```python
# Reset response messages
self.response_messages = []                            # Line 560

# Prepare in-context messages (load from DB + add new user messages)
current_in_context_messages, new_in_context_messages = \
    await _prepare_in_context_messages_no_persist_async(  # Line 563-565
        input_messages,
        agent_state,
        self.message_manager,
        self.actor
    )

initial_messages = new_in_context_messages             # Line 567
in_context_messages = current_in_context_messages      # Line 568

# Initialize tool rules solver (enforces structured output)
tool_rules_solver = ToolRulesSolver(agent_state.tool_rules)  # Line 570

# Create LLM client (routes to correct provider)
llm_client = LLMClient.create(                         # Line 573-576
    provider_type=agent_state.llm_config.model_endpoint_type,
    put_inner_thoughts_first=True,
    actor=self.actor,
)

# Initialize tracking
stop_reason = None                                     # Line 578
job_update_metadata = None                             # Line 579
usage = LettaUsageStatistics()                         # Line 580
```

**Phase 2: Main Execution Loop (Lines 602-800+)**

```python
# Loop up to max_steps times (default 10)
for i in range(max_steps):                             # Line 602

    # Handle approval requests (if last message requires approval)
    if in_context_messages[-1].role == "approval":     # Line 603
        # ... approval handling logic (lines 604-650)
        pass

    else:
        # Normal step execution

        # 1. Check for cancellation
        if await self._check_run_cancellation():       # Line 653
            stop_reason = LettaStopReason(
                stop_reason=StopReasonType.cancelled.value
            )
            break

        # 2. Generate unique step ID
        step_id = generate_step_id()                   # Line 660
        step_start = get_utc_timestamp_ns()            # Line 661

        # 3. Log step to database (for observability)
        logged_step = await self.step_manager.log_step_async(  # Line 663-682
            actor=self.actor,
            agent_id=agent_state.id,
            provider_name=agent_state.llm_config.model_endpoint_type,
            model=agent_state.llm_config.model,
            usage=UsageStatistics(...),
            run_id=self.current_run_id,
            step_id=step_id,
            status=StepStatus.PENDING,
        )

        # 4. Rebuild system prompt if memory changed
        in_context_messages = await self._rebuild_memory_async(  # Line 685-690
            in_context_messages=in_context_messages,
            agent_state=agent_state,
            tool_rules_solver=tool_rules_solver,
        )

        # 5. Prepare LLM request
        request_data, response_data, current_in_context_messages, \
        new_in_context_messages, valid_tool_names = \
            await self._prepare_step_data(             # Line 693-700
                agent_state=agent_state,
                llm_client=llm_client,
                in_context_messages=in_context_messages,
                tool_rules_solver=tool_rules_solver,
                dry_run=dry_run,
            )

        # If dry_run, return request payload
        if dry_run:                                    # Line 702
            return request_data                        # Line 703

        # 6. Call LLM API
        response = await llm_client.create_chat_completion_async(  # Line 706-711
            messages=request_data["messages"],
            tools=request_data.get("tools"),
            tool_choice=request_data.get("tool_choice"),
            # ... other params
        )

        # 7. Update usage statistics
        usage.step_count += 1                          # Line 714
        if response.usage:                             # Line 715
            usage.completion_tokens += response.usage.completion_tokens
            usage.prompt_tokens += response.usage.prompt_tokens
            usage.total_tokens += response.usage.total_tokens

        # 8. Extract tool calls from response
        first_message = response.choices[0].message    # Line 720
        tool_calls = first_message.tool_calls or []    # Line 721
        reasoning_content = first_message.content      # Line 722

        # 9. Handle the AI response (execute tools, update memory)
        persisted_messages, should_continue, stop_reason = \
            await self._handle_ai_response(            # Line 725-740
                tool_calls=tool_calls,
                reasoning_content=reasoning_content,
                agent_state=agent_state,
                tool_rules_solver=tool_rules_solver,
                usage=usage,
                step_id=step_id,
                is_final_step=(i == max_steps - 1),
                run_id=run_id,
            )

        # 10. Accumulate new messages
        new_message_idx = len(initial_messages) if initial_messages else 0  # Line 743
        self.response_messages.extend(persisted_messages[new_message_idx:])  # Line 744
        new_in_context_messages.extend(persisted_messages[new_message_idx:])  # Line 745

        # 11. Update context window
        initial_messages = None                        # Line 747
        in_context_messages = current_in_context_messages + \
                              new_in_context_messages  # Line 748

        # 12. Check if we should continue
        if not should_continue:                        # Line 751
            break  # Agent finished (called send_message or hit error)

        # Otherwise, loop continues for next step

# End of loop

# Return final state
return (agent_state, new_in_context_messages, stop_reason, usage)  # Line 858
```

---

## Step-by-Step Breakdown

### Step 1: Load Agent State

**Code:** [`letta/agents/letta_agent.py:180-184`](../letta-repo/letta/agents/letta_agent.py#L180-L184)

```python
agent_state = await self.agent_manager.get_agent_by_id_async(
    agent_id=self.agent_id,
    include_relationships=["tools", "memory", "tool_exec_environment_variables", "sources"],
    actor=self.actor,
)
```

**What Happens:**
1. Queries database via `AgentManager`
2. Loads `Agent` ORM object from database
3. Joins related tables: `Tool`, `Block`, `EnvironmentVariable`, `Source`
4. Converts to `AgentState` (Pydantic model) for use in memory
5. Returns fully hydrated agent state

**Related Files:**
- [`letta/services/agent_manager.py`](../letta-repo/letta/services/agent_manager.py) - Manager implementation
- [`letta/orm/agent.py`](../letta-repo/letta/orm/agent.py) - ORM model
- [`letta/schemas/agent.py`](../letta-repo/letta/schemas/agent.py) - Pydantic schema

### Step 2: Prepare In-Context Messages

**Code:** [`letta/agents/letta_agent.py:563-567`](../letta-repo/letta/agents/letta_agent.py#L563-L567)

**Helper Function:** [`letta/agents/helpers.py`](../letta-repo/letta/agents/helpers.py)

```python
current_in_context_messages, new_in_context_messages = \
    await _prepare_in_context_messages_no_persist_async(
        input_messages,      # User's new messages
        agent_state,         # Agent configuration
        self.message_manager,  # For DB access
        self.actor           # Current user
    )
```

**What Happens:**
1. Fetches recent message history from database (based on context window size)
2. Includes system message (prompt + memory)
3. Adds user's new input messages (not yet persisted to DB)
4. Returns:
   - `current_in_context_messages`: Existing messages from DB
   - `new_in_context_messages`: New messages to add

**Context Window Management:**
- Default limit: 8192 tokens (configurable)
- If near limit, triggers summarization (see [Memory Management](./memory-management.md))

### Step 3: Initialize Tool Rules Solver

**Code:** [`letta/agents/letta_agent.py:570`](../letta-repo/letta/agents/letta_agent.py#L570)

```python
tool_rules_solver = ToolRulesSolver(agent_state.tool_rules)
```

**What It Does:**
- Enforces structured output constraints
- Example: "Always call send_message to respond to user"
- Generates prompt additions to guide LLM
- Validates tool calls match rules

**Related Files:**
- [`letta/helpers/__init__.py`](../letta-repo/letta/helpers/__init__.py) - ToolRulesSolver class
- [`letta/schemas/tool.py`](../letta-repo/letta/schemas/tool.py) - ToolRule schema

### Step 4: Create LLM Client

**Code:** [`letta/agents/letta_agent.py:573-576`](../letta-repo/letta/agents/letta_agent.py#L573-L576)

```python
llm_client = LLMClient.create(
    provider_type=agent_state.llm_config.model_endpoint_type,  # e.g., "openai", "anthropic"
    put_inner_thoughts_first=True,                             # Reasoning before tool calls
    actor=self.actor,
)
```

**Factory Pattern:**
- Routes to correct provider based on `provider_type`
- Supports: OpenAI, Anthropic, Google Vertex, Ollama, etc.
- See [LLM Integration Analysis](./llm-integration.md) for details

**Related Files:**
- [`letta/llm_api/llm_client.py`](../letta-repo/letta/llm_api/llm_client.py) - Factory
- [`letta/llm_api/openai_client.py`](../letta-repo/letta/llm_api/openai_client.py) - OpenAI impl
- [`letta/llm_api/anthropic_client.py`](../letta-repo/letta/llm_api/anthropic_client.py) - Claude impl

### Step 5: Main Execution Loop

**Code:** [`letta/agents/letta_agent.py:602-850`](../letta-repo/letta/agents/letta_agent.py#L602-L850)

```python
for i in range(max_steps):  # Default: 10 steps max
    # ... execution logic
```

**Loop Iterations:**
- Each iteration = 1 "step" (one LLM call + tool executions)
- Agent continues until:
  - Calls `send_message` tool (signals completion)
  - Reaches `max_steps` limit
  - Error occurs
  - Run is cancelled

**Example Multi-Step Execution:**

```
Step 0: User asks "What's 2+2?"
  → Agent thinks: "I need to respond"
  → Calls: send_message("The answer is 4")
  → Loop ends (send_message called)

Step 0: User asks "Search web for Python news, then summarize"
  → Agent thinks: "I need to search first"
  → Calls: web_search("Python news")
  → Continue to step 1

Step 1: Agent receives search results
  → Agent thinks: "Now I can summarize"
  → Calls: send_message("Here's the latest Python news...")
  → Loop ends
```

### Step 6: Rebuild Memory (If Changed)

**Code:** [`letta/agents/letta_agent.py:685-690`](../letta-repo/letta/agents/letta_agent.py#L685-L690)

**Base Implementation:** [`letta/agents/base_agent.py:86-193`](../letta-repo/letta/agents/base_agent.py#L86-L193)

```python
in_context_messages = await self._rebuild_memory_async(
    in_context_messages=in_context_messages,
    agent_state=agent_state,
    tool_rules_solver=tool_rules_solver,
)
```

**When This Happens:**
- After tools modify memory blocks (e.g., `core_memory_replace`)
- System message needs updating with new memory content

**Process:**
1. Reload memory blocks from database (line 99)
2. Generate new system message with updated memory (line 165-173)
3. Compare with current system message (line 175-178)
4. If changed, update system message in DB (line 180-186)
5. Return updated message list

**Performance Optimization:**
- Only rebuilds if memory actually changed (string comparison)
- Avoids unnecessary DB writes

### Step 7: Call LLM API

**Code:** [`letta/agents/letta_agent.py:706-711`](../letta-repo/letta/agents/letta_agent.py#L706-L711)

```python
response = await llm_client.create_chat_completion_async(
    messages=request_data["messages"],          # Formatted conversation history
    tools=request_data.get("tools"),            # Available tool definitions
    tool_choice=request_data.get("tool_choice"),  # Force/allow/none
    # Additional provider-specific params...
)
```

**Request Structure:**
```json
{
  "messages": [
    {"role": "system", "content": "System prompt with memory..."},
    {"role": "user", "content": "User message"},
    {"role": "assistant", "content": "...", "tool_calls": [...]},
    {"role": "tool", "content": "Tool result"}
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "send_message",
        "description": "Send a message to the user",
        "parameters": {...}
      }
    }
  ]
}
```

**Response Structure:**
```python
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Thinking: I should respond...",  # Reasoning
        "tool_calls": [                             # Tool invocations
          {
            "id": "call_123",
            "type": "function",
            "function": {
              "name": "send_message",
              "arguments": '{"message": "Hello!"}'
            }
          }
        ]
      }
    }
  ],
  "usage": {
    "prompt_tokens": 1500,
    "completion_tokens": 50,
    "total_tokens": 1550
  }
}
```

### Step 8: Handle AI Response

**Code:** [`letta/agents/letta_agent.py:1635-1857`](../letta-repo/letta/agents/letta_agent.py#L1635-L1857)

This is a critical 223-line method that processes the LLM response:

```python
async def _handle_ai_response(
    self,
    tool_calls: list[ToolCall],                    # Line 1637
    reasoning_content: list[...],                  # Line 1638 (thinking)
    agent_state: AgentState,                       # Line 1639
    tool_rules_solver: ToolRulesSolver,            # Line 1640
    usage: LettaUsageStatistics,                   # Line 1641
    step_id: str,                                  # Line 1642
    initial_messages: list[Message] | None = None, # Line 1643
    is_final_step: bool = False,                   # Line 1644
    step_metrics: StepMetrics | None = None,       # Line 1645
    run_id: str | None = None,                     # Line 1646
    is_approval: bool | None = None,               # Line 1647
    is_denial: bool = False,                       # Line 1648
    denial_reason: str | None = None,              # Line 1649
) -> tuple[list[Message], bool, LettaStopReason | None]:
    """
    Process LLM response, execute tools, persist messages.

    Returns:
      - persisted_messages: All messages saved to DB
      - should_continue: Whether to continue loop
      - stop_reason: Why execution stopped (if stopped)
    """
```

**Processing Flow:**

```python
# 1. Parse tool calls (lines 1651-1690)
for tool_call in tool_calls:
    tool_name = tool_call.function.name
    tool_args = json.loads(tool_call.function.arguments)

    # Validate tool exists
    if tool_name not in valid_tools:
        # Create error message
        continue

    # Check if tool requires approval
    if tool_requires_approval(tool_name):
        # Create approval request message
        # Return early with should_continue=False
        return (approval_messages, False, None)

# 2. Execute all tool calls (lines 1692-1750)
tool_results = []
for tool_call in tool_calls:
    # Execute in sandbox (LOCAL/E2B/MODAL)
    result = await self.tool_execution_manager.execute_tool_async(
        agent_state=agent_state,
        tool_name=tool_call.function.name,
        tool_args=tool_call.function.arguments,
        actor=self.actor,
    )
    tool_results.append(result)

    # Check if tool modified memory
    if tool_name == "core_memory_replace" or tool_name == "memory":
        # Mark that memory changed (will trigger rebuild)
        memory_was_updated = True

# 3. Create messages for DB persistence (lines 1752-1800)
messages_to_save = []

# Add assistant's reasoning/thinking message
if reasoning_content:
    assistant_msg = Message(
        role="assistant",
        content=reasoning_content,
        tool_calls=tool_calls,
        step_id=step_id,
    )
    messages_to_save.append(assistant_msg)

# Add tool result messages
for tool_call, result in zip(tool_calls, tool_results):
    tool_msg = Message(
        role="tool",
        content=result.output,
        tool_call_id=tool_call.id,
        name=tool_call.function.name,
        step_id=step_id,
    )
    messages_to_save.append(tool_msg)

# 4. Persist to database (lines 1802-1820)
persisted_messages = await self.message_manager.create_many_messages_async(
    messages=messages_to_save,
    agent_id=agent_state.id,
    actor=self.actor,
)

# 5. Determine if should continue (lines 1822-1840)
should_continue = True

# Check if send_message was called (signals completion)
for tool_call in tool_calls:
    if tool_call.function.name == "send_message":
        should_continue = False
        stop_reason = LettaStopReason(stop_reason=StopReasonType.end_turn.value)
        break

# Check if this was the last allowed step
if is_final_step and should_continue:
    should_continue = False
    stop_reason = LettaStopReason(stop_reason=StopReasonType.max_steps.value)

# Return results
return (persisted_messages, should_continue, stop_reason)
```

**Key Decisions in This Method:**

1. **Approval Requests**: If a tool requires approval, execution pauses
2. **Memory Updates**: Detects when memory was modified
3. **Continuation Logic**: Determines if loop should continue
4. **Error Handling**: Tool errors are captured and returned to LLM

---

## Tool Execution

**Deep Dive:** See [Tool System Analysis](./tool-system.md) for full details

**Location:** [`letta/services/tool_executor/tool_execution_manager.py`](../letta-repo/letta/services/tool_executor/tool_execution_manager.py)

### Core Tools

**Location:** [`letta/functions/function_sets/base.py`](../letta-repo/letta/functions/function_sets/base.py)

1. **`send_message`** - Send response to user (signals completion)
2. **`memory`** - Unified memory interface (read/write memory blocks)
3. **`core_memory_replace`** - Legacy memory editing
4. **`core_memory_append`** - Legacy memory appending
5. **`archival_memory_search`** - Search long-term memory
6. **`archival_memory_insert`** - Add to long-term memory

### Tool Execution Sandboxes

**Configuration:** Set via `tool_sandbox` setting

1. **LOCAL** - Execute in subprocess with venv isolation
2. **E2B** - Execute in E2B cloud sandbox
3. **MODAL** - Execute in Modal serverless environment

**Code Reference:** [`letta/sandbox/`](../letta-repo/letta/sandbox/)

---

## Memory Updates

### When Memory Changes

**Trigger Points:**

1. **Tool Calls:**
   - `memory()` function
   - `core_memory_replace()`
   - `core_memory_append()`

2. **Block Manager:**
   ```python
   await self.block_manager.update_block_async(
       block_id=block.id,
       value=new_value,
       actor=self.actor,
   )
   ```

3. **System Prompt Rebuild:**
   - After memory update, `_rebuild_memory_async()` is called
   - System message updated with new memory content
   - Next LLM call sees updated memory

**Memory Flow Example:**

```
Step 1: User says "My name is John"
  → Agent thinks: "I should remember this"
  → Calls: memory({
      "block": "human",
      "old_content": "Name: unknown",
      "new_content": "Name: John. Likes coding."
    })
  → BlockManager updates database
  → _rebuild_memory_async() updates system message

Step 2: Agent continues
  → Sees updated memory in system prompt
  → Calls: send_message("Nice to meet you, John!")
```

---

## Streaming vs Non-Streaming

### Non-Streaming: `step()`

**Returns:** Complete `LettaResponse` after all steps finish

```python
response = await agent.step(input_messages=[...])
# Returns after agent completes all thinking and tool execution
```

**Use Case:** Simpler API, batch processing

### Streaming: `step_stream()` and `step_stream_no_tokens()`

**Location:**
- [`letta/agents/letta_agent.py:892-1632`](../letta-repo/letta/agents/letta_agent.py#L892-L1632) - Full token streaming
- [`letta/agents/letta_agent.py:208-549`](../letta-repo/letta/agents/letta_agent.py#L208-L549) - Step streaming (no tokens)

**Returns:** AsyncGenerator yielding events

```python
async for chunk in agent.step_stream(input_messages=[...]):
    # Yields events as they happen:
    # - Token chunks (if token streaming enabled)
    # - Tool calls
    # - Tool results
    # - Final response
    print(chunk)
```

**Event Types:**

1. **Token Chunks** (if `stream_tokens=True`):
   ```json
   {"type": "chat_completion_chunk", "delta": {"content": "Hello"}}
   ```

2. **Messages**:
   ```json
   {"message_type": "assistant_message", "content": [...], "tool_calls": [...]}
   ```

3. **Stop Reason**:
   ```json
   {"stop_reason": "end_turn"}
   ```

4. **Usage Statistics**:
   ```json
   {"completion_tokens": 42, "prompt_tokens": 1500}
   ```

**Implementation Details:**

- Uses Python AsyncGenerators (`yield`)
- Server-Sent Events (SSE) format for HTTP streaming
- Interfaces: `OpenAIStreamingInterface`, `AnthropicStreamingInterface`

**Related Files:**
- [`letta/interfaces/openai_streaming_interface.py`](../letta-repo/letta/interfaces/openai_streaming_interface.py)
- [`letta/interfaces/anthropic_streaming_interface.py`](../letta-repo/letta/interfaces/anthropic_streaming_interface.py)

---

## Error Handling

### Error Types

1. **Context Window Exceeded**
   ```python
   # letta/agents/letta_agent.py (various locations)
   raise ContextWindowExceededError(
       f"Context window exceeded: {current_tokens} > {max_tokens}"
   )
   ```
   **Handling:** Triggers summarization or fails gracefully

2. **Tool Execution Errors**
   ```python
   # letta/services/tool_executor/tool_execution_manager.py
   try:
       result = execute_tool(...)
   except Exception as e:
       # Capture error and return to LLM
       result = ToolExecutionResult(
           status="error",
           output=f"Error: {str(e)}"
       )
   ```
   **Handling:** Error message returned to LLM, which can retry or adapt

3. **LLM API Errors**
   ```python
   # letta/llm_api/llm_client.py
   try:
       response = await provider.create_chat_completion(...)
   except RateLimitError:
       # Exponential backoff retry
   except APIError as e:
       # Log and re-raise
   ```
   **Handling:** Retries with backoff, logs to telemetry

4. **Run Cancellation**
   ```python
   # letta/agents/letta_agent.py:653-660
   if await self._check_run_cancellation():
       stop_reason = LettaStopReason(
           stop_reason=StopReasonType.cancelled.value
       )
       break
   ```
   **Handling:** Clean shutdown, partial results saved

### Telemetry and Observability

**OpenTelemetry Integration:**

```python
# letta/agents/letta_agent.py
@trace_method  # Decorator adds tracing
async def step(...):
    span = tracer.start_span("agent_step")
    span.set_attributes({"agent_id": self.agent_id})
    # ... execution
    span.end()
```

**Metrics Tracked:**
- Token usage per step
- Latency per step
- Tool execution counts
- Error rates

**Related Files:**
- [`letta/otel/tracing.py`](../letta-repo/letta/otel/tracing.py) - Tracing utilities
- [`letta/otel/metric_registry.py`](../letta-repo/letta/otel/metric_registry.py) - Metrics

---

## Summary

### Execution Flow Diagram

```
User Input
    ↓
step() [Line 169]
    ↓
Load Agent State [Line 180-184]
    ↓
_step() [Line 551]
    ↓
┌───────────────────────────────────┐
│ Execution Loop (max_steps times) │
│ [Line 602-850]                    │
├───────────────────────────────────┤
│  1. Check Cancellation            │
│  2. Generate Step ID              │
│  3. Rebuild Memory (if changed)   │
│  4. Prepare LLM Request           │
│  5. Call LLM API                  │
│  6. Extract Tool Calls            │
│  7. _handle_ai_response()         │
│     ├─ Execute Tools              │
│     ├─ Persist Messages           │
│     └─ Check Continue             │
│  8. Update Context                │
│  9. Break if done                 │
└───────────────────────────────────┘
    ↓
Format Response [Line 199-205]
    ↓
Return LettaResponse
```

### Key Files Reference

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| Base Agent | `letta/agents/base_agent.py` | 1-202 | Abstract base class |
| Main Agent | `letta/agents/letta_agent.py` | 77-1904 | Core implementation |
| Entry Point | `letta/agents/letta_agent.py` | 169-205 | Public `step()` method |
| Core Logic | `letta/agents/letta_agent.py` | 551-862 | Internal `_step()` method |
| Response Handler | `letta/agents/letta_agent.py` | 1635-1857 | `_handle_ai_response()` |
| Memory Rebuild | `letta/agents/base_agent.py` | 86-193 | `_rebuild_memory_async()` |
| Helpers | `letta/agents/helpers.py` | - | Utility functions |

### Next Steps

- **Memory Management**: See [memory-management.md](./memory-management.md)
- **Tool System**: See [tool-system.md](./tool-system.md)
- **API Layer**: See [api-layer.md](./api-layer.md)
- **Streaming**: See [streaming-system.md](./streaming-system.md)

---

**Navigation Tips:**

To view a specific line in the code:
1. Open the file: `letta-repo/letta/agents/letta_agent.py`
2. Jump to line number using your editor (Ctrl+G in most editors)
3. Or use GitHub: `https://github.com/letta-ai/letta/blob/main/letta/agents/letta_agent.py#L169`

**Code Exploration Commands:**

```bash
# View specific lines
sed -n '169,205p' letta-repo/letta/agents/letta_agent.py

# Search for method definitions
grep -n "async def" letta-repo/letta/agents/letta_agent.py

# Find all references to a function
grep -r "_handle_ai_response" letta-repo/letta/
```
