---
name: Session and State Management
description: ADK session management, state handling with magic prefixes, memory services, and persistence strategies. Use when implementing conversation state, user preferences, or cross-session memory.
version: 1.0.0
---

# ADK Session and State Management (December 2025)

## Overview

ADK provides sophisticated state management across multiple scopes:
- **Session State** - Single conversation context
- **User State** - Persists across all user sessions
- **App State** - Global across all users and sessions
- **Memory Service** - Long-term intelligent memory

## Session Lifecycle

### Creating Sessions

```python
from google.adk.sessions import InMemorySessionService

session_service = InMemorySessionService()

# Create new session
session = session_service.create_session(
    app_name="soundsage",
    user_id="user_123",
    initial_state={
        "mode": "intern",
        "context": "mixing",
        "user:name": "Producer Mike",
        "user:preferred_genre": "electronic",
        "app:version": "2.0.0"
    }
)

print(session.id)  # Unique session ID
```

### Retrieving Sessions

```python
# Get existing session
session = session_service.get_session(
    app_name="soundsage",
    user_id="user_123",
    session_id="abc123"
)

# Get or create pattern
session = session_service.get_or_create_session(
    app_name="soundsage",
    user_id="user_123",
    initial_state={"mode": "intern"}
)
```

### Listing Sessions

```python
# List all sessions for a user
sessions = session_service.list_sessions(
    app_name="soundsage",
    user_id="user_123"
)

for s in sessions:
    print(f"{s.id}: {s.state.get('mode')}")
```

## State Magic Prefixes

### Scope Types

| Prefix | Scope | Persistence |
|--------|-------|-------------|
| (none) | Session | Lost when session ends |
| `user:` | User | Persists across all user's sessions |
| `app:` | Application | Persists across all sessions globally |

### Usage Examples

```python
# Session-scoped (default) - temporary
session.state["current_page"] = "mixing"
session.state["undo_stack"] = []
session.state["temp_selection"] = ["track-1"]

# User-scoped - preferences and history
session.state["user:name"] = "Producer Mike"
session.state["user:dark_mode"] = True
session.state["user:favorite_presets"] = ["warm_tape", "punchy"]
session.state["user:recent_projects"] = ["project_a", "project_b"]

# App-scoped - global configuration
session.state["app:version"] = "2.0.0"
session.state["app:maintenance_mode"] = False
session.state["app:feature_flags"] = {"new_eq": True}
```

## Template Variables in Instructions

Inject state dynamically into agent instructions:

```python
agent = Agent(
    name="personalized_assistant",
    model="gemini-2.5-flash",
    instruction="""
    You are helping {user:name}, a {user:preferred_genre} music producer.

    Current session context:
    - Mode: {mode}
    - View: {current_page}
    - Selected tracks: {selected_tracks}

    App version: {app:version}

    Adjust your communication style based on mode:
    - intern: Brief, action-focused
    - mentor: Educational, explain reasoning
    """,
    tools=[...]
)
```

**How It Works**:
1. Before each LLM call, ADK replaces `{key}` with `session.state[key]`
2. Supports nested access: `{user:preferences.theme}`
3. Missing keys render as empty string
4. Complex objects serialize to JSON

## State Updates via Events

State changes are tracked through events:

```python
from google.adk.events import Event, EventAction

# Create event with state changes
event = Event(
    author="agent",
    content="Updated your preferences",
    action=EventAction(
        state_delta={
            "user:dark_mode": True,
            "last_action": "toggle_theme"
        }
    )
)

# Append to session (atomically updates state)
session.append_event(event)

# State is now updated
print(session.state["user:dark_mode"])  # True
```

## SessionService Implementations

### InMemorySessionService (Development)

```python
from google.adk.sessions import InMemorySessionService

session_service = InMemorySessionService()

# Fast, no persistence
# Good for development and testing
# Data lost on restart
```

### DatabaseSessionService (Production)

```python
from google.adk.sessions import DatabaseSessionService

# SQLite
session_service = DatabaseSessionService(
    connection_string="sqlite:///sessions.db"
)

# PostgreSQL
session_service = DatabaseSessionService(
    connection_string="postgresql://user:pass@host:5432/db"
)

# MySQL
session_service = DatabaseSessionService(
    connection_string="mysql://user:pass@host:3306/db"
)
```

### VertexAISessionService (Enterprise)

```python
from google.adk.sessions import VertexAISessionService

session_service = VertexAISessionService(
    project_id="my-project",
    location="us-central1"
)

# Fully managed
# Auto-scaling
# Built-in security
```

## Memory Service (Long-Term Memory)

For persistent, intelligent memory across sessions:

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

**Memory Bank Features**:
- Topic-based memory organization
- Gemini-powered memory extraction
- Intelligent retrieval based on context
- Google Research ACL 2025 paper implementation

### Memory vs State

| Aspect | State | Memory |
|--------|-------|--------|
| Structure | Key-value | Unstructured/semantic |
| Retrieval | Exact key | Context-based search |
| Use Case | Settings, preferences | Facts, knowledge |
| Persistence | Scoped by prefix | Permanent |

## Pydantic State Models

Type-safe state with Pydantic:

```python
from pydantic import BaseModel
from typing import Optional, List

class UserPreferences(BaseModel):
    name: str
    dark_mode: bool = True
    preferred_genre: str = "electronic"
    favorite_presets: List[str] = []

class SessionContext(BaseModel):
    mode: str = "intern"
    current_view: str = "mixer"
    selected_tracks: List[str] = []
    undo_stack: List[dict] = []

class DAWState(BaseModel):
    ui: UIState
    mixer: MixerState
    transport: TransportState
    project: ProjectState
    monitoring: MonitoringState
    agent_mode: str = "intern"

# Usage
state = DAWState(**session.state)
state.transport.bpm = 128
session.state.update(state.model_dump())
```

## SoundSage State Pattern

From ss-agent implementation:

```python
# State structure for DAW
initial_state = {
    # Session-scoped (temporary)
    "ui": {
        "currentView": "mixer",
        "selectedTrackIds": [],
        "playheadPosition": 0
    },
    "transport": {
        "isPlaying": False,
        "bpm": 120,
        "timeSignature": {"numerator": 4, "denominator": 4}
    },

    # User-scoped (persistent preferences)
    "user:preferences": {
        "dark_mode": True,
        "metering_standard": "lufs"
    },
    "user:recent_presets": [],

    # App-scoped (global)
    "app:version": "2.0.0"
}

# Mode switching via state
def switch_mode(session, new_mode: str):
    session.state["mode"] = new_mode
    # Agent instruction template {mode} will reflect change
```

## State-Based Decision Making

Use state in tool logic:

```python
def session_tool(action: str, **params) -> dict:
    """Session management tool."""

    if action == "set_mode":
        new_mode = params.get("mode", "intern")
        return {
            "action": "session.set_mode",
            "state_update": {"mode": new_mode},
            "status": "success",
            "message": f"Switched to {new_mode} mode"
        }

    if action == "undo":
        # Access undo stack from state
        return {
            "action": "session.undo",
            "status": "success"
        }
```

## Best Practices

### 1. Namespace State Keys
```python
# Good - clear ownership
"user:preferences.theme"
"mixer:tracks.selected"
"transport:bpm"

# Avoid - ambiguous
"theme"
"selected"
"bpm"
```

### 2. Minimize State Size
```python
# Good - reference by ID
"selectedTrackIds": ["track-1", "track-2"]

# Avoid - storing full objects
"selectedTracks": [{full track object...}]
```

### 3. Use Defaults
```python
# Safe access with defaults
mode = session.state.get("mode", "intern")
tracks = session.state.get("selectedTrackIds", [])
```

### 4. Atomic Updates
```python
# Use state_delta for atomic updates
event = Event(
    action=EventAction(
        state_delta={
            "user:last_action": "volume_change",
            "user:action_count": session.state.get("user:action_count", 0) + 1
        }
    )
)
```

### 5. Clear Sensitive State
```python
# On session end, clear temporary data
def cleanup_session(session):
    keys_to_clear = [k for k in session.state if not k.startswith(("user:", "app:"))]
    for key in keys_to_clear:
        del session.state[key]
```

## State Migration

When evolving state schema:

```python
def migrate_state(session):
    """Migrate state to new schema version."""
    version = session.state.get("app:state_version", 1)

    if version < 2:
        # v1 → v2: Rename preference keys
        if "user:darkMode" in session.state:
            session.state["user:dark_mode"] = session.state.pop("user:darkMode")
        session.state["app:state_version"] = 2

    if version < 3:
        # v2 → v3: Add new required fields
        session.state.setdefault("user:notifications", True)
        session.state["app:state_version"] = 3
```

## Debugging State

```python
# Log state for debugging
import json

def log_state(session):
    print("=== Session State ===")
    for key, value in sorted(session.state.items()):
        scope = "session"
        if key.startswith("user:"):
            scope = "user"
        elif key.startswith("app:"):
            scope = "app"
        print(f"[{scope}] {key}: {json.dumps(value)[:100]}")
```
