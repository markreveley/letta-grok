# Tool Calling (Function Calling)

## What is Tool Calling?

**Tool calling** (also called "function calling") is how AI agents perform actions and interact with external systems. Instead of just generating text, agents can call predefined functions to:
- Send messages
- Update memory
- Search databases
- Execute code
- Make API calls
- And much more

## The Core Concept

### Before Tool Calling

```
User: "What's 127 * 83?"
Agent: "Let me calculate that... I believe it's 10,541"
      [Wrong! Agent guessed]
```

### With Tool Calling

```
User: "What's 127 * 83?"
Agent: [Calls calculator tool]
Tool: calculator(127, 83) → 10,541
Agent: "The answer is 10,541"
      [Correct! Used actual calculator]
```

---

## How It Works

### 1. Tools are Defined

```python
def calculator(expression: str) -> float:
    """
    Evaluate a mathematical expression.
    
    Args:
        expression: Math expression like "127 * 83"
    
    Returns:
        The numerical result
    """
    return eval(expression)  # (simplified example)
```

### 2. LLM Sees Tool Schema

```json
{
  "name": "calculator",
  "description": "Evaluate a mathematical expression",
  "parameters": {
    "type": "object",
    "properties": {
      "expression": {
        "type": "string",
        "description": "Math expression like '127 * 83'"
      }
    },
    "required": ["expression"]
  }
}
```

### 3. LLM Decides to Call Tool

```
LLM Response:
{
  "reasoning": "User wants calculation, I should use calculator",
  "tool_calls": [
    {
      "name": "calculator",
      "arguments": {"expression": "127 * 83"}
    }
  ]
}
```

### 4. System Executes Tool

```python
# Letta executes the function
result = calculator("127 * 83")
# result = 10541
```

### 5. Result Returned to LLM

```
Tool Result:
{
  "tool_call_id": "call_123",
  "output": "10541"
}
```

### 6. LLM Formulates Response

```
LLM Final Response:
{
  "tool_calls": [
    {
      "name": "send_message",
      "arguments": {
        "message": "The answer is 10,541"
      }
    }
  ]
}
```

---

## Built-in Letta Tools

### Core Communication Tools

**1. send_message** - Send response to user

```python
send_message(message: str)
```

**Example:**
```
Agent: "I've found the answer"
[Calls: send_message("The answer is 42")]
User sees: "The answer is 42"
```

### Memory Tools

**2. memory** - Update memory blocks

```python
memory(
    command: str,          # "str_replace", "insert", "create", "delete"
    path: str,             # Block label
    old_str: str = None,   # Text to replace
    new_str: str = None,   # Replacement text
    ...
)
```

**Example:**
```
User: "My email is john@example.com"
Agent: [Calls: memory(
    command="str_replace",
    path="human",
    old_str="Email: unknown",
    new_str="Email: john@example.com"
)]
```

**3. archival_memory_insert** - Store in long-term memory

```python
archival_memory_insert(
    content: str,
    tags: List[str] = None
)
```

**4. archival_memory_search** - Search long-term memory

```python
archival_memory_search(
    query: str,
    top_k: int = 5
)
```

### Utility Tools

**5. conversation_search** - Search conversation history

```python
conversation_search(
    query: str,
    roles: List[str] = None,  # ["user", "assistant", "tool"]
    limit: int = 10
)
```

**6. web_search** - Search the web

```python
web_search(query: str)
```

**7. run_code** - Execute code safely

```python
run_code(
    code: str,
    language: str = "python"
)
```

---

## Multi-Step Tool Use

Agents can chain multiple tool calls:

```
User: "Research quantum computing and save key points"

Agent Step 1:
  [Calls: web_search("quantum computing basics")]
  Result: [article summaries...]

Agent Step 2:
  [Calls: archival_memory_insert(
    content="Quantum computing uses qubits..."
  )]
  Result: "Saved successfully"

Agent Step 3:
  [Calls: send_message(
    "I've researched quantum computing and saved..."
  )]
```

---

## Tool Execution Flow

```
┌─────────────────────────────────────────┐
│ 1. User sends message                   │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 2. LLM generates response with tool call│
│    {                                    │
│      "tool_calls": [{                   │
│        "name": "calculator",            │
│        "arguments": {"expr": "2+2"}     │
│      }]                                 │
│    }                                    │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 3. Letta validates tool call            │
│    - Tool exists?                       │
│    - Arguments valid?                   │
│    - Agent has permission?              │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 4. Execute tool in sandbox              │
│    result = calculator("2+2")           │
│    # result = 4                         │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 5. Return result to LLM                 │
│    {                                    │
│      "role": "tool",                    │
│      "content": "4",                    │
│      "name": "calculator"               │
│    }                                    │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 6. LLM decides next action              │
│    - Call another tool?                 │
│    - Or send_message to user?           │
└─────────────────────────────────────────┘
```

---

## Custom Tools

### Creating a Custom Tool

```python
def get_weather(location: str, units: str = "celsius") -> str:
    """
    Get current weather for a location.
    
    Args:
        location: City name or coordinates
        units: Temperature units (celsius or fahrenheit)
    
    Returns:
        Weather description
    """
    # Your implementation
    api_response = weather_api.get(location)
    return f"Weather in {location}: {api_response.temp}°{units[0].upper()}"

# Register with Letta
tool = client.tools.create(
    name="get_weather",
    source_code=inspect.getsource(get_weather),
    source_type="python"
)

# Attach to agent
client.agents.tools.attach(
    agent_id=agent.id,
    tool_id=tool.id
)
```

### Using Your Custom Tool

```
User: "What's the weather in Tokyo?"

Agent:
  [Calls: get_weather(location="Tokyo", units="celsius")]
  Result: "Weather in Tokyo: 22°C, Sunny"
  
  [Calls: send_message("It's currently 22°C and sunny in Tokyo")]
```

---

## Tool Rules & Constraints

### Structured Output

Force agent to ALWAYS call specific tools:

```python
agent = client.agents.create(
    tool_rules=[
        {
            "type": "require_tool",
            "tool_name": "send_message",
            "description": "Always use send_message to respond to user"
        }
    ]
)
```

**Result:**
```
Agent CANNOT respond without calling send_message
→ Ensures consistent response format
→ Prevents agent from "thinking out loud"
```

### Tool Approval

Require human approval for sensitive tools:

```python
agent = client.agents.create(
    tools_with_approval_required=["delete_database", "send_email"]
)
```

**Flow:**
```
Agent: "I should delete old data"
[Calls: delete_database("old_data")]

→ Execution PAUSED
→ User receives approval request
→ User approves/denies
→ If approved, tool executes
→ If denied, tool skipped
```

---

## Tool Sandboxing

For security, tools execute in isolated environments:

### Local Sandbox

```python
# Code runs in subprocess with timeout
result = run_code("""
print("Hello from sandbox!")
""")
# Output: "Hello from sandbox!"
```

**Limits:**
- Time limit (30 seconds default)
- Memory limit
- No network access
- No file system access (except designated paths)

### Cloud Sandboxes

**E2B (E2B.dev):**
```python
settings.tool_sandbox = "E2B"
# Tools execute in cloud containers
# Complete isolation
# Auto-scaling
```

**Modal (modal.com):**
```python
settings.tool_sandbox = "MODAL"
# Serverless function execution
# Pay-per-use
# Infinite scale
```

---

## Advanced: MCP Tools

Letta supports Model Context Protocol (MCP) for external tool integration:

```python
# Add MCP server
client.mcp_servers.create(
    name="weather-server",
    config={...}
)

# List available tools
tools = client.tools.list_mcp_tools_by_server("weather-server")

# Add tool to agent
tool = client.tools.add_mcp_tool(
    mcp_server_name="weather-server",
    mcp_tool_name="get_forecast"
)

client.agents.tools.attach(agent_id, tool.id)
```

Now agent can call tools from external MCP server!

---

## Best Practices

### ✅ DO: Provide Clear Descriptions

```python
def search_database(query: str) -> List[dict]:
    """
    Search the product database using full-text search.
    
    Args:
        query: Search terms (e.g., "laptop 16GB RAM")
        
    Returns:
        List of matching products with name, price, specs
    
    Examples:
        search_database("gaming laptop") → [...]
        search_database("budget phone") → [...]
    """
```

### ✅ DO: Return Structured Results

```python
# Good
return {
    "success": True,
    "results": [{"name": "Product 1", "price": 999}],
    "count": 1
}

# Bad  
return "Found 1 product: Product 1 for $999"
```

### ✅ DO: Handle Errors Gracefully

```python
def risky_operation(param: str) -> dict:
    try:
        result = external_api.call(param)
        return {"success": True, "data": result}
    except Exception as e:
        return {"success": False, "error": str(e)}
```

### ❌ DON'T: Make Tools Too Complex

```python
# Bad - too many parameters
def complex_tool(a, b, c, d, e, f, g, h):
    ...

# Better - split into multiple tools
def simple_tool_1(a, b):
    ...
def simple_tool_2(c, d):
    ...
```

---

## Common Patterns

### Pattern 1: Search → Process → Respond

```
1. search_database(query="product")
2. analyze_results(results)
3. send_message("Here's what I found...")
```

### Pattern 2: Validate → Execute → Confirm

```
1. validate_input(data)
2. execute_action(data)
3. send_message("Action completed")
```

### Pattern 3: Iterative Refinement

```
1. initial_search(broad_query)
2. If insufficient: refined_search(narrow_query)
3. If still insufficient: ask_user_for_clarification()
4. final_response()
```

---

## Debugging Tools

### Check Tool Execution

```python
# Get run details
run = client.runs.get(run_id)

# See all tool calls in run
for step in run.steps:
    print(f"Tool: {step.tool_name}")
    print(f"Args: {step.arguments}")
    print(f"Result: {step.result}")
```

### Tool Execution Logs

```python
# Enable detailed logging
import logging
logging.basicConfig(level=logging.DEBUG)

# See tool execution traces
# Shows:
# - Tool validation
# - Argument parsing
# - Execution time
# - Results
```

---

## Summary

**Tool Calling** enables agents to:
- ✅ Perform actions (not just talk)
- ✅ Access external systems and APIs
- ✅ Modify their own memory
- ✅ Execute code safely
- ✅ Chain multiple operations

**Key Tools in Letta:**
- `send_message` - Respond to user
- `memory` - Update memory blocks
- `archival_memory_*` - Long-term storage
- `run_code` - Execute code
- `web_search` - Search internet
- Custom tools - Your own functions

**Tool Flow:**
1. Agent decides to use tool
2. Letta validates and executes
3. Result returned to agent
4. Agent continues or responds

**Code References:**
- Built-in Tools: [`letta/functions/function_sets/base.py`](../letta-repo/letta/functions/function_sets/base.py)
- Tool Manager: [`letta/services/tool_manager.py`](../letta-repo/letta/services/tool_manager.py)
- Tool Executor: [`letta/services/tool_executor/`](../letta-repo/letta/services/tool_executor/)

**Related Concepts:**
- [Memory Blocks](./04-memory-blocks.md) - Updated via memory() tool
- [Agent Lifecycle](./08-agent-lifecycle.md) - When tools are called
- [Sandboxing](./15-sandboxing.md) - Safe tool execution
