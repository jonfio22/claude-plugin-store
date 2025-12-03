---
name: Tool Development
description: ADK tool creation patterns including FunctionTool, OpenAPI tools, MCP integration, AgentTool, and human-in-the-loop confirmation. Use when creating new agent tools or integrating external APIs.
version: 1.0.0
---

# ADK Tool Development (December 2025)

## Overview

ADK tools enable agents to interact with external systems. Tool categories:
1. **FunctionTool** - Python functions as tools
2. **OpenAPI Tools** - Auto-generated from OpenAPI specs
3. **MCP Tools** - Model Context Protocol integration
4. **Built-in Tools** - Google Search, Code Execution
5. **AgentTool** - Other agents as tools
6. **Third-Party** - LangChain, LlamaIndex tools

## FunctionTool

The most common tool type:

```python
from google.adk.tools import FunctionTool

def get_weather(location: str, units: str = "celsius") -> str:
    """Get current weather for a location.

    Args:
        location: City name or coordinates
        units: Temperature units (celsius/fahrenheit)

    Returns:
        Weather description with temperature
    """
    # Implementation
    return f"Weather in {location}: 22°C, sunny"

# Create tool from function
weather_tool = FunctionTool(get_weather)

# Use in agent
agent = Agent(
    name="weather_agent",
    model="gemini-2.5-flash",
    tools=[weather_tool]
)
```

**Critical Requirements**:
1. Type hints for all parameters
2. Comprehensive docstring with Args section
3. Return type hint
4. Descriptive function name

### Human-in-the-Loop Confirmation

Require user confirmation before execution:

```python
def delete_file(file_path: str) -> str:
    """Delete a file from the system."""
    os.remove(file_path)
    return f"Deleted {file_path}"

# Boolean confirmation
tool = FunctionTool(delete_file, require_confirmation=True)

# Custom confirmation criteria
def needs_confirmation(context) -> bool:
    # Only confirm for certain paths
    return "/important/" in context.tool_input.get("file_path", "")

tool = FunctionTool(delete_file, require_confirmation=needs_confirmation)
```

### Async Functions

```python
async def fetch_data(url: str) -> dict:
    """Fetch data from URL asynchronously."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

tool = FunctionTool(fetch_data)
```

## SoundSage Mega-Tool Pattern

From ss-agent - consolidate operations into mega-tools:

```python
def tracks(
    action: str,
    selector: str | list[str] | None = None,
    **params
) -> dict:
    """Control track properties in the mixer.

    Args:
        action: Operation to perform - one of:
            volume, pan, mute, solo, arm, name, color, output,
            add, remove, duplicate, reorder
        selector: Track selector - can be:
            - "all" - All tracks
            - "selected" - Currently selected tracks
            - "armed" - Record-armed tracks
            - "muted" / "soloed" - Tracks in that state
            - "bus" / "audio" - Tracks by type
            - "group:Drums" - Tracks in named group
            - ["id1", "id2"] - Explicit track IDs
            - "track-name" - Single track by name
        **params: Action-specific parameters
            volume: value (0.0-1.0)
            pan: value (-1.0 to 1.0)
            mute/solo/arm: state (bool)
            name: new_name (str)
            color: hex_color (str)
            output: bus_id (str)

    Returns:
        Action dict with action, params, status, message
    """
    if action not in VALID_ACTIONS:
        return {"status": "error", "message": f"Unknown action: {action}"}

    return {
        "action": f"tracks.{action}",
        "params": {"selector": selector, **params},
        "status": "success",
        "message": f"Applied {action} to {selector}"
    }
```

**Benefits**:
- 84% tool consolidation (62 → 8 tools)
- Batch operations via selectors
- Consistent return format
- Easier prompt engineering

## OpenAPI Tools

Auto-generate tools from OpenAPI v3 specs:

```python
from google.adk.tools import OpenAPIToolset

# From URL
github_tools = OpenAPIToolset(
    spec_url="https://raw.githubusercontent.com/github/rest-api-description/main/openapi.yaml",
    authentication={
        "type": "bearer",
        "token": os.environ["GITHUB_TOKEN"]
    }
)

# From file
stripe_tools = OpenAPIToolset(
    spec_path="./stripe-openapi.yaml",
    authentication={
        "type": "api_key",
        "header": "Authorization",
        "value": f"Bearer {os.environ['STRIPE_API_KEY']}"
    }
)

# Use in agent
agent = Agent(
    name="github_agent",
    model="gemini-2.5-flash",
    tools=github_tools.get_tools()
)
```

## MCP Tools (Model Context Protocol)

Integrate MCP servers:

```python
from google.adk.tools import MCPToolset

# stdio transport
mcp_tools = MCPToolset(
    server_command="python",
    server_args=["/path/to/mcp_server.py"],
    transport="stdio"
)

# SSE transport
mcp_tools = MCPToolset(
    server_url="http://localhost:3000/sse",
    transport="sse"
)

# WebSocket transport
mcp_tools = MCPToolset(
    server_url="ws://localhost:3000/ws",
    transport="websocket"
)

agent = Agent(
    name="mcp_agent",
    tools=mcp_tools.get_tools()
)
```

### MCP Toolbox for Databases

Pre-built MCP server for 30+ databases:

```python
# Connect to database via MCP Toolbox
db_tools = MCPToolset(
    server_command="npx",
    server_args=["@anthropic/mcp-toolbox", "--database", "postgresql://..."],
    transport="stdio"
)
```

Supports: BigQuery, AlloyDB, PostgreSQL, MySQL, SQL Server, and more.

## Built-in Tools

### Google Search
```python
from google.adk.tools import google_search

agent = Agent(
    model="gemini-2.5-flash",
    tools=[google_search]
)
```

### Code Execution
```python
from google.adk.tools import code_execution

agent = Agent(
    model="gemini-2.5-flash",
    tools=[code_execution],
    instruction="You can write and execute Python code."
)
```

### AgentEngineSandboxCodeExecutor (New 2025)
```python
from google.adk.tools import AgentEngineSandboxCodeExecutor

# Persistent sandbox up to 100MB
sandbox = AgentEngineSandboxCodeExecutor(
    project_id="my-project",
    location="us-central1"
)

agent = Agent(
    model="gemini-2.5-flash",
    tools=[sandbox]
)
```

## AgentTool

Use other agents as tools:

```python
from google.adk.tools import AgentTool

specialist_agent = LlmAgent(
    name="sql_expert",
    model="gemini-2.5-flash",
    instruction="You are an SQL expert. Generate optimized queries."
)

# Wrap agent as tool
sql_tool = AgentTool(specialist_agent)

coordinator = Agent(
    name="data_coordinator",
    model="gemini-2.5-flash",
    instruction="Use the SQL expert for database queries.",
    tools=[sql_tool]
)
```

## Third-Party Tool Integration

### LangChain Tools
```python
from google.adk.tools.langchain_tool import LangChainTool
from langchain_community.tools import WikipediaQueryRun

wiki = WikipediaQueryRun()
wiki_tool = LangChainTool(langchain_tool=wiki)

agent = Agent(tools=[wiki_tool])
```

### LlamaIndex Tools
```python
from google.adk.tools.llama_index_tool import LlamaIndexTool
from llama_index.tools import QueryEngineTool

query_tool = QueryEngineTool.from_defaults(query_engine=index.as_query_engine())
llama_tool = LlamaIndexTool(llama_index_tool=query_tool)

agent = Agent(tools=[llama_tool])
```

## Tool Design Best Practices

### 1. Clear Documentation
```python
def process_audio(
    track_id: str,
    effect_type: str,
    settings: dict
) -> dict:
    """Apply audio effect to a track.

    This function applies the specified effect with given settings
    to the audio track. Changes are non-destructive and can be undone.

    Args:
        track_id: Unique identifier for the target track
        effect_type: Effect to apply - one of:
            - "eq" - Equalizer
            - "compressor" - Dynamic range compression
            - "reverb" - Reverb effect
        settings: Effect-specific parameters as dict:
            EQ: {"frequency": 1000, "gain": 3, "q": 1.4}
            Compressor: {"threshold": -20, "ratio": 4}
            Reverb: {"decay": 2.0, "mix": 0.3}

    Returns:
        Dict with keys:
            - status: "success" or "error"
            - message: Human-readable result description
            - applied_settings: Actual settings used

    Raises:
        ValueError: If track_id not found or invalid effect_type

    Example:
        >>> process_audio("track-1", "eq", {"frequency": 200, "gain": -3})
        {"status": "success", "message": "Applied EQ to track-1"}
    """
```

### 2. Input Validation
```python
def safe_tool(value: int) -> str:
    """Process a value safely."""
    if not isinstance(value, int):
        return json.dumps({"error": f"Expected int, got {type(value)}"})
    if value < 0 or value > 100:
        return json.dumps({"error": "Value must be 0-100"})

    # Safe to process
    return json.dumps({"result": value * 2})
```

### 3. Consistent Return Format
```python
# Standard response structure
def tool_response(status: str, message: str, data: dict = None) -> dict:
    return {
        "status": status,  # "success" or "error"
        "message": message,
        "data": data or {}
    }
```

### 4. Error Handling
```python
def robust_tool(param: str) -> str:
    """A tool that handles errors gracefully."""
    try:
        result = risky_operation(param)
        return json.dumps({"status": "success", "result": result})
    except ValidationError as e:
        return json.dumps({"status": "error", "message": f"Invalid input: {e}"})
    except ExternalAPIError as e:
        return json.dumps({"status": "error", "message": f"API unavailable: {e}"})
    except Exception as e:
        return json.dumps({"status": "error", "message": f"Unexpected error: {e}"})
```

### 5. Presets Pattern (from ss-agent)
```python
COMPRESSOR_PRESETS = {
    "gentle": {"threshold": -20, "ratio": 2, "attack": 30, "release": 200},
    "punchy": {"threshold": -15, "ratio": 4, "attack": 5, "release": 100},
    "aggressive": {"threshold": -10, "ratio": 8, "attack": 1, "release": 50},
}

def effects(
    action: str,
    track_id: str,
    preset: str | None = None,
    **raw_params
) -> dict:
    """Apply effect with optional preset."""
    if preset:
        params = COMPRESSOR_PRESETS.get(preset, {})
        params.update(raw_params)  # Override with any raw params
    else:
        params = raw_params

    return apply_effect(action, track_id, params)
```

## Testing Tools

```python
import pytest
from google.adk.tools import ToolCall

def test_weather_tool():
    tool = FunctionTool(get_weather)

    # Test tool call
    call = ToolCall(
        name="get_weather",
        input={"location": "San Francisco", "units": "celsius"}
    )

    result = tool.execute(call)

    assert "San Francisco" in result
    assert "°C" in result

def test_validation():
    tool = FunctionTool(bounded_value)

    # Test invalid input
    call = ToolCall(name="bounded_value", input={"value": 150})
    result = tool.execute(call)

    assert "error" in result.lower()
```
