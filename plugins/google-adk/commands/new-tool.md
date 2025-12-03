---
name: new-tool
description: Create a new ADK tool (FunctionTool, mega-tool, or OpenAPI integration)
allowed_tools: [Read, Write, Edit, Glob, Grep]
---

# Create New ADK Tool

You are creating a new tool for a Google ADK agent. Follow the project's mega-tool patterns from `/Users/fiorante/Documents/sound-sage-finalboss/ss-agent/tools/`.

## Gather Information

Ask the user for:
1. **Tool Type**:
   - `function` - Simple FunctionTool
   - `mega` - Mega-tool with multiple actions (recommended)
   - `openapi` - Generated from OpenAPI spec
2. **Tool Name** - snake_case identifier
3. **Actions/Operations** - What the tool does
4. **Parameters** - Required and optional params
5. **Target Agent** - Which agent gets this tool

## Mega-Tool Template (Recommended)

Based on SoundSage pattern:

```python
"""
{Tool Name} mega-tool for {purpose}.
"""
from typing import Literal
import json

VALID_ACTIONS = [
    "action1",
    "action2",
    "action3",
]

# Optional: Presets for common configurations
PRESETS = {
    "action1": {
        "preset_a": {"param1": "value1", "param2": "value2"},
        "preset_b": {"param1": "value3", "param2": "value4"},
    }
}


def {tool_name}(
    action: Literal[{action_literals}],
    target: str | None = None,
    preset: str | None = None,
    **params
) -> dict:
    """{Tool description}.

    Args:
        action: Operation to perform - one of: {actions_list}
        target: Target identifier (if applicable)
        preset: Optional preset name for common configurations
        **params: Action-specific parameters:
            action1: param_a (type), param_b (type)
            action2: param_c (type)

    Returns:
        Dict with keys:
            - action: "{tool_name}.{action}" format
            - params: Applied parameters
            - status: "success" or "error"
            - message: Human-readable result

    Examples:
        >>> {tool_name}("action1", target="item-1", param_a=10)
        {{"action": "{tool_name}.action1", "params": {{"param_a": 10}}, "status": "success"}}

        >>> {tool_name}("action2", preset="preset_a")
        {{"action": "{tool_name}.action2", "params": {{...}}, "status": "success"}}
    """
    # Validate action
    if action not in VALID_ACTIONS:
        return {
            "status": "error",
            "message": f"Unknown action: {action}. Valid: {VALID_ACTIONS}"
        }

    # Resolve preset if provided
    final_params = {}
    if preset and action in PRESETS:
        preset_params = PRESETS[action].get(preset)
        if preset_params:
            final_params.update(preset_params)
        else:
            return {
                "status": "error",
                "message": f"Unknown preset: {preset}. Valid: {list(PRESETS[action].keys())}"
            }

    # Override with explicit params
    final_params.update(params)

    # Action-specific validation
    if action == "action1":
        # Validate action1-specific params
        pass
    elif action == "action2":
        # Validate action2-specific params
        pass

    return {
        "action": f"{tool_name}.{action}",
        "params": {"target": target, **final_params},
        "status": "success",
        "message": f"Applied {action}" + (f" to {target}" if target else "")
    }
```

## Simple FunctionTool Template

```python
from google.adk.tools import FunctionTool


def {function_name}(
    param1: str,
    param2: int = 10,
    param3: bool = False
) -> str:
    """{Function description}.

    Args:
        param1: Description of param1
        param2: Description of param2 (default: 10)
        param3: Description of param3 (default: False)

    Returns:
        JSON string with result

    Example:
        >>> {function_name}("value", param2=20)
        '{{"status": "success", "result": ...}}'
    """
    try:
        # Implementation
        result = do_something(param1, param2, param3)

        return json.dumps({
            "status": "success",
            "result": result
        })
    except ValueError as e:
        return json.dumps({
            "status": "error",
            "message": f"Invalid input: {e}"
        })
    except Exception as e:
        return json.dumps({
            "status": "error",
            "message": f"Unexpected error: {e}"
        })


# Create tool instance
{function_name}_tool = FunctionTool({function_name})
```

## OpenAPI Tool Template

```python
from google.adk.tools import OpenAPIToolset
import os


# Generate tools from OpenAPI spec
{api_name}_tools = OpenAPIToolset(
    spec_url="{openapi_spec_url}",  # or spec_path for local file
    authentication={
        "type": "bearer",  # or "api_key", "basic"
        "token": os.environ["{API_KEY_ENV_VAR}"]
    }
)

# Get individual tools
tools_list = {api_name}_tools.get_tools()
```

## Test Template

```python
import pytest
from {package}.tools.{tool_name} import {tool_name}


class Test{ToolName}:
    """Tests for {tool_name} mega-tool."""

    def test_action1_success(self):
        """Test successful action1."""
        result = {tool_name}(action="action1", target="item-1", param_a=10)

        assert result["status"] == "success"
        assert result["action"] == "{tool_name}.action1"
        assert result["params"]["param_a"] == 10

    def test_action1_with_preset(self):
        """Test action1 with preset."""
        result = {tool_name}(action="action1", preset="preset_a")

        assert result["status"] == "success"
        assert "param1" in result["params"]

    def test_invalid_action(self):
        """Test invalid action returns error."""
        result = {tool_name}(action="invalid")

        assert result["status"] == "error"
        assert "Unknown action" in result["message"]

    def test_invalid_preset(self):
        """Test invalid preset returns error."""
        result = {tool_name}(action="action1", preset="nonexistent")

        assert result["status"] == "error"
        assert "Unknown preset" in result["message"]

    @pytest.mark.parametrize("action", [
        "action1",
        "action2",
        "action3",
    ])
    def test_all_actions_valid(self, action):
        """Test all valid actions work."""
        result = {tool_name}(action=action)
        assert result["status"] == "success"
```

## Integration

After creating the tool, add to agent:

```python
# In tools/__init__.py
from .{tool_name} import {tool_name}

CORE_TOOLS = [
    {tool_name},
    # ... other tools
]

# Or if using FunctionTool wrapper
from google.adk.tools import FunctionTool

CORE_TOOLS = [
    FunctionTool({tool_name}),
    # ... other tools
]
```

## Checklist

- [ ] Type hints on all parameters
- [ ] Comprehensive docstring with Args, Returns, Examples
- [ ] Input validation with clear error messages
- [ ] Presets for common configurations (if mega-tool)
- [ ] Consistent return format
- [ ] Tests for all actions and edge cases
- [ ] Added to CORE_TOOLS in __init__.py
- [ ] Documented in agent's system prompt
