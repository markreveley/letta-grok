# Tool System - Deep Dive

Comprehensive analysis of Letta's tool execution system with code references.

## Overview

Letta's tool system allows agents to interact with external systems, update memory, and perform actions. It supports multiple tool sources and execution environments.

**Primary Files:**
- [`letta/services/tool_manager.py`](../letta-repo/letta/services/tool_manager.py) - Tool CRUD (1,043 lines)
- [`letta/services/tool_executor/tool_execution_manager.py`](../letta-repo/letta/services/tool_executor/tool_execution_manager.py) - Execution orchestration (160 lines)
- [`letta/functions/function_sets/`](../letta-repo/letta/functions/function_sets/) - Built-in tools

---

## Tool Types

### Location: [`letta/schemas/enums.py`](../letta-repo/letta/schemas/enums.py)

```python
class ToolType(str, Enum):
    LETTA_CORE = "letta_core"                    # Core tools (send_message, memory)
    LETTA_MEMORY_CORE = "letta_memory_core"      # Memory-specific tools
    LETTA_MULTI_AGENT_CORE = "multi_agent"       # Multi-agent communication
    LETTA_BUILTIN = "letta_builtin"              # Built-in tools (web_search, run_code)
    LETTA_FILES_CORE = "letta_files_core"        # File operations
    EXTERNAL_MCP = "external_mcp"                # MCP (Model Context Protocol) tools
    CUSTOM = "custom"                            # User-defined Python tools
```

---

## Built-in Tools

### Core Tools

**Location:** [`letta/functions/function_sets/base.py`](../letta-repo/letta/functions/function_sets/base.py)

1. **`send_message(message: str)`** - Send response to user (signals completion)
2. **`memory(...)`** - Unified memory management interface
3. **`conversation_search(query, ...)`** - Search conversation history
4. **`archival_memory_search(query, top_k)`** - Search long-term memory
5. **`archival_memory_insert(content, tags)`** - Add to archival memory

### Multi-Agent Tools

**Location:** [`letta/functions/function_sets/multi_agent.py`](../letta-repo/letta/functions/function_sets/multi_agent.py)

1. **`send_message_to_agent_and_wait_for_reply(...)`** - Direct agent communication
2. **`send_message_to_agents_matching_tags(...)`** - Broadcast to agent groups

### File Tools

**Location:** [`letta/functions/function_sets/files.py`](../letta-repo/letta/functions/function_sets/files.py)

1. **`open_files(paths)`** - Open files into memory
2. **`close_files(paths)`** - Close files
3. **`grep_files(pattern, ...)`** - Search file contents
4. **`semantic_search_files(query, ...)`** - Vector search in files

### Built-in Tools

**Location:** [`letta/functions/function_sets/builtin.py`](../letta-repo/letta/functions/function_sets/builtin.py)

1. **`web_search(query)`** - Search the web
2. **`run_code(code, language)`** - Execute code in sandbox
3. **`fetch_webpage(url)`** - Fetch webpage content

---

## Tool Execution Flow

### Entry Point: ToolExecutionManager

**Location:** [`letta/services/tool_executor/tool_execution_manager.py:94-160`](../letta-repo/letta/services/tool_executor/tool_execution_manager.py#L94-L160)

```python
class ToolExecutionManager:
    @trace_method
    async def execute_tool_async(
        self,
        function_name: str,                          # Line 96
        function_args: dict,
        tool: Tool,
        step_id: str | None = None
    ) -> ToolExecutionResult:
        """Execute a tool asynchronously and persist any state changes."""

        # 1. Get appropriate executor based on tool type
        executor = ToolExecutorFactory.get_executor(
            tool_type=tool.tool_type,
            message_manager=self.message_manager,
            agent_manager=self.agent_manager,
            block_manager=self.block_manager,
            run_manager=self.run_manager,
            passage_manager=self.passage_manager,
            actor=self.actor,
        )

        # 2. Execute tool
        try:
            result = await executor.execute_async(
                function_name=function_name,
                function_args=function_args,
                tool=tool,
                agent_state=self.agent_state,
                step_id=step_id,
            )
            return result
        except Exception as e:
            # 3. Handle errors gracefully
            return ToolExecutionResult(
                status="error",
                output=get_friendly_error_msg(e),
                traceback=traceback.format_exc(),
            )
```

### Tool Executor Factory

**Location:** [`letta/services/tool_executor/tool_execution_manager.py:32-65`](../letta-repo/letta/services/tool_executor/tool_execution_manager.py#L32-L65)

```python
class ToolExecutorFactory:
    """Factory for creating appropriate tool executors based on tool type."""

    _executor_map: Dict[ToolType, Type[ToolExecutor]] = {
        ToolType.LETTA_CORE: LettaCoreToolExecutor,
        ToolType.LETTA_MEMORY_CORE: LettaCoreToolExecutor,
        ToolType.LETTA_MULTI_AGENT_CORE: LettaMultiAgentToolExecutor,
        ToolType.LETTA_BUILTIN: LettaBuiltinToolExecutor,
        ToolType.LETTA_FILES_CORE: LettaFileToolExecutor,
        ToolType.EXTERNAL_MCP: ExternalMCPToolExecutor,
    }

    @classmethod
    def get_executor(cls, tool_type: ToolType, ...) -> ToolExecutor:
        """Get the appropriate executor for the given tool type."""
        executor_class = cls._executor_map.get(tool_type, SandboxToolExecutor)
        return executor_class(...)
```

---

## Tool Manager

**Location:** [`letta/services/tool_manager.py:189-1043`](../letta-repo/letta/services/tool_manager.py#L189-L1043)

### Create Custom Tool

```python
async def create_tool_async(
    self,
    tool_create: ToolCreate,
    actor: User
) -> Tool:
    """
    Create a new tool from source code.

    Process:
    1. Parse source code to extract function signature
    2. Generate JSON schema from signature
    3. Validate schema
    4. Store tool in database
    """
```

### Attach Tool to Agent

```python
async def attach_tool_to_agent_async(
    self,
    agent_id: str,
    tool_id: str,
    actor: User
) -> None:
    """
    Attach a tool to an agent.

    Creates association in database (many-to-many relationship).
    """
```

---

## Sandbox Execution

### Sandbox Types

**Configuration:** `settings.tool_sandbox`

1. **LOCAL** - Execute in subprocess with venv isolation
2. **E2B** - Execute in E2B cloud sandbox
3. **MODAL** - Execute in Modal serverless environment

### SandboxToolExecutor

**Location:** [`letta/services/tool_executor/sandbox_tool_executor.py`](../letta-repo/letta/services/tool_executor/sandbox_tool_executor.py)

```python
class SandboxToolExecutor(ToolExecutor):
    async def execute_async(
        self,
        function_name: str,
        function_args: dict,
        tool: Tool,
        agent_state: AgentState,
        step_id: str,
    ) -> ToolExecutionResult:
        """Execute custom Python tool in sandbox."""

        # 1. Prepare sandbox environment
        sandbox_config = SandboxConfig(
            type=settings.tool_sandbox,
            timeout=tool.timeout or 30,
        )

        # 2. Execute code
        if sandbox_config.type == "LOCAL":
            result = await self._execute_local(tool.source_code, function_args)
        elif sandbox_config.type == "E2B":
            result = await self._execute_e2b(tool.source_code, function_args)
        elif sandbox_config.type == "MODAL":
            result = await self._execute_modal(tool.source_code, function_args)

        # 3. Return result
        return ToolExecutionResult(
            status="success",
            output=result,
        )
```

---

## Tool Approval System

### Configuration

Agents can be configured to require approval before executing specific tools.

**Location:** [`letta/schemas/agent.py`](../letta-repo/letta/schemas/agent.py)

```python
class AgentState:
    tools_with_approval_required: List[str] = Field(
        default_factory=list,
        description="Tools that require approval before execution"
    )
```

### Approval Flow

```python
# Agent wants to call a tool
if tool_name in agent.tools_with_approval_required:
    # Create approval request message
    approval_message = Message(
        role="approval",
        content=f"Request approval to call {tool_name}",
        tool_calls=[tool_call],
    )

    # Return to user for approval
    # Execution pauses here
    return LettaResponse(
        messages=[approval_message],
        stop_reason="requires_approval",
    )

# On next user message
if user_approves:
    # Execute tool
    result = await tool_execution_manager.execute_tool_async(...)
else:
    # Skip tool execution
    result = ToolExecutionResult(
        status="denied",
        output="User denied approval"
    )
```

---

## MCP (Model Context Protocol) Tools

### Location: [`letta/services/tool_executor/mcp_tool_executor.py`](../letta-repo/letta/services/tool_executor/mcp_tool_executor.py)

Letta acts as an MCP client, integrating tools from MCP servers.

```python
class ExternalMCPToolExecutor(ToolExecutor):
    async def execute_async(
        self,
        function_name: str,
        function_args: dict,
        tool: Tool,
        agent_state: AgentState,
        step_id: str,
    ) -> ToolExecutionResult:
        """Execute MCP tool via RPC."""

        # 1. Connect to MCP server
        mcp_client = MCPClient(tool.mcp_server_config)
        await mcp_client.connect()

        # 2. Call tool on MCP server
        result = await mcp_client.call_tool(
            tool_name=function_name,
            arguments=function_args,
        )

        # 3. Return result
        return ToolExecutionResult(
            status="success",
            output=result,
        )
```

---

## Tool Schema Generation

**Location:** [`letta/helpers/tool_schema_generator.py`](../letta-repo/letta/helpers/tool_schema_generator.py)

Letta automatically generates JSON schemas from Python function signatures.

```python
def memory(
    agent_state: "AgentState",
    command: str,
    path: Optional[str] = None,
    old_str: Optional[str] = None,
    new_str: Optional[str] = None,
) -> Optional[str]:
    """
    Memory management tool.

    Args:
        command: The sub-command to execute
        path: Path to the memory block
        old_str: Old text to replace
        new_str: New text to replace with
    """
    pass

# Generated schema:
{
  "type": "function",
  "function": {
    "name": "memory",
    "description": "Memory management tool.",
    "parameters": {
      "type": "object",
      "properties": {
        "command": {"type": "string", "description": "The sub-command to execute"},
        "path": {"type": "string", "description": "Path to the memory block"},
        "old_str": {"type": "string", "description": "Old text to replace"},
        "new_str": {"type": "string", "description": "New text to replace with"}
      },
      "required": ["command"]
    }
  }
}
```

---

## Tool Return Value Limits

**Constant:** `FUNCTION_RETURN_VALUE_TRUNCATED = 2048` (characters)

**Location:** [`letta/constants.py`](../letta-repo/letta/constants.py)

Tool outputs are truncated to prevent context window overflow.

```python
# In tool execution
if len(result_output) > FUNCTION_RETURN_VALUE_TRUNCATED:
    result_output = result_output[:FUNCTION_RETURN_VALUE_TRUNCATED]
    result_output += f"\n[Output truncated at {FUNCTION_RETURN_VALUE_TRUNCATED} chars]"
```

---

## Summary

### Tool Execution Pipeline

```
LLM Response
    ↓
Extract Tool Calls
    ↓
ToolExecutionManager.execute_tool_async()
    ↓
ToolExecutorFactory.get_executor(tool_type)
    ↓
┌─────────────────────────────────────────┐
│ Route to Appropriate Executor           │
├─────────────────────────────────────────┤
│ • LettaCoreToolExecutor                 │
│ • LettaBuiltinToolExecutor              │
│ • LettaMultiAgentToolExecutor           │
│ • LettaFileToolExecutor                 │
│ • ExternalMCPToolExecutor               │
│ • SandboxToolExecutor (custom)          │
└────────────────┬────────────────────────┘
                 ↓
        Execute in Sandbox
                 ↓
         Return Result
                 ↓
    ToolExecutionResult
                 ↓
     Add to Message History
                 ↓
      Next Agent Step
```

### Key Files Reference

| Component | File | Purpose |
|-----------|------|---------|
| Tool Manager | `letta/services/tool_manager.py` | CRUD operations |
| Execution Manager | `letta/services/tool_executor/tool_execution_manager.py` | Orchestration |
| Core Tools | `letta/functions/function_sets/base.py` | Built-in core tools |
| Multi-Agent Tools | `letta/functions/function_sets/multi_agent.py` | Agent communication |
| File Tools | `letta/functions/function_sets/files.py` | File operations |
| Built-in Tools | `letta/functions/function_sets/builtin.py` | Web search, code execution |
| MCP Executor | `letta/services/tool_executor/mcp_tool_executor.py` | MCP integration |
| Sandbox Executor | `letta/services/tool_executor/sandbox_tool_executor.py` | Custom tool execution |

---

**See also:**
- [Agent Execution](./agent-execution.md) - How tools fit in execution loop
- [Memory Management](./memory-management.md) - Memory tool details
- [API Layer](./api-layer.md) - Tool management endpoints
