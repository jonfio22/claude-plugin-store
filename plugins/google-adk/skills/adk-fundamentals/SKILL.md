---
name: ADK Fundamentals
description: Core Google Agent Development Kit concepts including agents, runner, sessions, events, state management, memory, and artifacts. Use when building new ADK agents, understanding ADK architecture, or debugging agent behavior.
version: 1.0.0
---

# Google ADK Fundamentals (December 2025)

## Overview

Google's Agent Development Kit (ADK) is an open-source, code-first framework for building AI agents. Introduced at Google Cloud NEXT 2025, ADK is:
- **Model-agnostic**: Works with Gemini, Claude, GPT-4, Llama, and 100+ models via LiteLLM
- **Deployment-agnostic**: Local, Cloud Run, Vertex AI Agent Engine, GKE
- **Framework-compatible**: Integrates with LangChain, LlamaIndex, CrewAI

**Current Version**: Python ADK v1.0.0+ (requires Python 3.10+)

## Installation

```bash
# Stable release
pip install google-adk

# Development version (latest features)
pip install git+https://github.com/google/adk-python.git@main
```

## Core Architecture Components

### 1. Agent

The fundamental building block. Three types:

**LlmAgent (most common)**:
```python
from google.adk.agents import Agent

root_agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    description="What this agent does (for routing)",
    instruction="Detailed behavior instructions",
    tools=[tool1, tool2],
    sub_agents=[child_agent1, child_agent2]
)
```

**Key Agent Parameters**:
- `name`: Unique identifier (required)
- `model`: LLM model string or Model object (required)
- `description`: Used for agent routing decisions
- `instruction`: System prompt defining behavior
- `tools`: List of available tools
- `sub_agents`: Child agents for delegation
- `generate_content_config`: Temperature, top_p, etc.

### 2. Runner

The execution engine that orchestrates agent workflows:

```python
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService

session_service = InMemorySessionService()
runner = Runner(
    agent=root_agent,
    session_service=session_service,
    memory_service=memory_service  # optional
)

# Synchronous execution
result = runner.run("User message", session=session)

# Asynchronous execution
result = await runner.run_async("User message", session=session)
```

### 3. Session

Represents a conversation thread:

```python
session = session_service.create_session(
    app_name="my_app",
    user_id="user_123",
    initial_state={
        "theme": "dark",
        "user:name": "Alice",  # Persists across user sessions
        "app:version": "2.0"   # Persists across all sessions
    }
)
```

**State Magic Prefixes**:
- No prefix: Session-scoped (lost when session ends)
- `user:` prefix: Persists across all user's sessions
- `app:` prefix: Persists across all application sessions

**Template Variables** - inject state into instructions:
```python
instruction="""
Hello {user:name}, your theme is {theme}.
App version: {app:version}
"""
```

### 4. Events

The communication mechanism between components:

```python
from google.adk.events import Event, EventAction

# Events have:
# - author: "user", "agent", "tool", "system"
# - content: Text or multimodal content
# - action: EventAction with state_delta, tool_calls, etc.

event = Event(
    author="user",
    content="What's the weather?",
    action=EventAction(state_delta={"last_query": "weather"})
)
```

### 5. SessionService Implementations

**Development**:
```python
from google.adk.sessions import InMemorySessionService
session_service = InMemorySessionService()
```

**Production (SQL)**:
```python
from google.adk.sessions import DatabaseSessionService
session_service = DatabaseSessionService(
    connection_string="postgresql://user:pass@host/db"
)
```

**Production (Vertex AI)**:
```python
from google.adk.sessions import VertexAISessionService
session_service = VertexAISessionService(
    project_id="my-project",
    location="us-central1"
)
```

### 6. Memory Service

For long-term memory across sessions:

```python
from google.adk.memory import VertexAIMemoryBankService

memory_service = VertexAIMemoryBankService(
    project_id="my-project",
    location="us-central1"
)

runner = Runner(
    agent=root_agent,
    session_service=session_service,
    memory_service=memory_service
)
```

### 7. Artifact Service

Handle binary data (images, PDFs, audio):

```python
from google.adk.artifacts import GcsArtifactService

artifact_service = GcsArtifactService(
    bucket_name="my-artifacts-bucket"
)
```

## Model Configuration

**Direct String (simplest)**:
```python
agent = Agent(model="gemini-2.5-flash", ...)
```

**Vertex AI (production)**:
```python
from google.adk.models.vertexai import VertexAI

agent = Agent(
    model=VertexAI(
        model="gemini-2.5-flash",
        project="my-project",
        location="us-central1"
    )
)
```

**Third-Party via LiteLLM**:
```python
from google.adk.models.lite_llm import LiteLlm
import os

os.environ["ANTHROPIC_API_KEY"] = "sk-..."
agent = Agent(model=LiteLlm(model="anthropic/claude-3-sonnet-20240229"))
```

## Generation Config

Control model behavior:

```python
from google.genai import types

agent = Agent(
    name="precise_agent",
    model="gemini-2.5-flash",
    generate_content_config=types.GenerateContentConfig(
        temperature=0.2,  # Lower = more deterministic
        top_p=0.9,
        max_output_tokens=2048
    )
)
```

## Environment Setup

**Google AI Studio (development)**:
```bash
export GOOGLE_API_KEY="your-api-key"
```

**Vertex AI (production)**:
```bash
gcloud auth application-default login
export GOOGLE_CLOUD_PROJECT="my-project"
export GOOGLE_CLOUD_LOCATION="us-central1"
```

## SoundSage-Specific Patterns

Based on the ss-agent implementation:

**Dual-Mode Agent Pattern**:
```python
def get_generation_config(mode: str):
    if mode == "intern":
        return types.GenerateContentConfig(temperature=0.3, top_p=0.9)
    else:  # mentor
        return types.GenerateContentConfig(temperature=0.5, top_p=0.95)

root_agent = Agent(
    name="soundsage_ninja",
    model="gemini-2.5-flash",
    instruction=get_full_instruction(mode),
    generate_content_config=get_generation_config(mode),
    tools=CORE_TOOLS
)
```

**Standard Export Pattern** (for ADK discovery):
```python
# __init__.py
from .agent import root_agent, ALL_TOOLS

__all__ = ['root_agent', 'ALL_TOOLS']
```

## Best Practices

1. **Start with gemini-2.5-flash** - Fast, cost-effective for most agents
2. **Use InMemorySessionService for dev** - Switch to DB/Vertex for production
3. **Template variables** - Inject dynamic state into instructions
4. **Magic prefixes** - Use `user:` and `app:` for cross-session persistence
5. **Pydantic models** - Type your state for safety

## Common Patterns

**Agent with State Injection**:
```python
agent = Agent(
    instruction="""
    You are helping {user:name}.
    Current context: {context}
    Mode: {mode}
    """,
    ...
)
```

**Session Initialization**:
```python
session = session_service.create_session(
    app_name="soundsage",
    user_id=user_id,
    initial_state={
        "mode": "intern",
        "user:preferences": {},
        "context": "mixing session"
    }
)
```
