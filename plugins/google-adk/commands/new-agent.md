---
name: new-agent
description: Scaffold a new Google ADK agent with proper structure, prompts, and configuration
allowed_tools: [Read, Write, Edit, Glob, Bash]
---

# Create New ADK Agent

You are scaffolding a new Google ADK agent. Follow the project's established patterns from `/Users/fiorante/Documents/sound-sage-finalboss/ss-agent`.

## Gather Information

Ask the user for:
1. **Agent Name** - kebab-case identifier (e.g., "mixing-assistant")
2. **Agent Purpose** - What the agent does
3. **Model** - gemini-2.5-flash (default) or gemini-2.5-pro
4. **Tools Needed** - Which tools the agent will use
5. **Location** - Where to create (default: alongside ss-agent)

## Directory Structure to Create

```
{agent-name}/
├── pyproject.toml
├── .env.example
├── {agent_name}/
│   ├── __init__.py
│   ├── agent.py
│   ├── server.py
│   ├── prompts/
│   │   ├── __init__.py
│   │   └── system.py
│   ├── tools/
│   │   ├── __init__.py
│   │   └── {tool_name}.py
│   └── state/
│       ├── __init__.py
│       └── models.py
└── tests/
    ├── __init__.py
    └── test_tools.py
```

## Template Files

### pyproject.toml
```toml
[project]
name = "{agent-name}"
version = "0.1.0"
description = "{agent purpose}"
requires-python = ">=3.10"
dependencies = [
    "google-adk>=1.19.0",
    "google-genai>=1.0.0",
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "pydantic>=2.0.0",
    "python-dotenv>=1.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
]
```

### __init__.py (package root)
```python
from .agent import root_agent

__all__ = ['root_agent']
```

### agent.py
```python
from google.adk.agents import Agent
from google.genai import types
from .prompts.system import SYSTEM_PROMPT
from .tools import CORE_TOOLS

root_agent = Agent(
    name="{agent_name}",
    model="gemini-2.5-flash",
    description="{agent purpose}",
    instruction=SYSTEM_PROMPT,
    tools=CORE_TOOLS,
    generate_content_config=types.GenerateContentConfig(
        temperature=0.3,
        top_p=0.9,
    ),
)
```

### prompts/system.py
```python
SYSTEM_PROMPT = """
# {Agent Name}

You are {agent purpose}.

## Context Available

[Document state/context the agent receives]

## Tools Reference

[Document each tool briefly]

## Decision Rules

1. Act immediately when request is clear
2. Ask for clarification when ambiguous
3. [Domain-specific rules]

## Communication Style

[Define response format, length, tone]

## Example Interactions

**User**: [example input]
**Assistant**: [example response]
"""
```

### server.py
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from .agent import root_agent
import os

app = FastAPI(title="{Agent Name} API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

session_service = InMemorySessionService()
runner = Runner(agent=root_agent, session_service=session_service)

@app.get("/health")
async def health():
    return {"status": "healthy", "agent": root_agent.name}

@app.post("/api/run")
async def run_endpoint(request: dict):
    session = session_service.get_or_create_session(
        app_name=root_agent.name,
        user_id=request.get("user_id", "default")
    )
    result = await runner.run_async(request["message"], session=session)
    return {"response": result.content}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "server:app",
        host=os.getenv("HOST", "0.0.0.0"),
        port=int(os.getenv("PORT", 8000)),
        reload=True
    )
```

### .env.example
```
GOOGLE_API_KEY=your-api-key-here
PORT=8000
HOST=0.0.0.0
```

## After Scaffolding

1. Copy `.env.example` to `.env` and add your API key
2. Install: `pip install -e .`
3. Run locally: `adk web .` or `python -m {agent_name}.server`
4. Customize prompts in `prompts/system.py`
5. Add tools in `tools/` directory
6. Add tests in `tests/`

## Validation

After creating, verify:
- [ ] `adk web .` starts without errors
- [ ] Health endpoint responds
- [ ] Agent responds to basic queries
