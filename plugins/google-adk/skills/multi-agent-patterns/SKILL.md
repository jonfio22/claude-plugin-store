---
name: Multi-Agent Patterns
description: Multi-agent orchestration patterns for Google ADK including Sequential, Parallel, Loop, hierarchical systems, agent transfer, and A2A protocol. Use when designing multi-agent architectures or implementing agent coordination.
version: 1.0.0
---

# ADK Multi-Agent Orchestration Patterns (December 2025)

## Overview

ADK supports sophisticated multi-agent systems through three primary mechanisms:
1. **Workflow Agents** - Deterministic execution patterns (Sequential, Parallel, Loop)
2. **LLM-Driven Delegation** - Dynamic agent transfer via LLM decisions
3. **Agent as Tool** - Use agents as callable tools while maintaining control
4. **A2A Protocol** - Cross-organization agent communication (v0.3)

## Workflow Agents

### SequentialAgent

Execute sub-agents in strict order:

```python
from google.adk.agents import SequentialAgent, LlmAgent

# Define specialized agents
code_writer = LlmAgent(
    name="code_writer",
    model="gemini-2.5-flash",
    instruction="Write clean, documented code based on requirements."
)

code_reviewer = LlmAgent(
    name="code_reviewer",
    model="gemini-2.5-flash",
    instruction="Review code for bugs, security, and best practices."
)

code_refactorer = LlmAgent(
    name="code_refactorer",
    model="gemini-2.5-flash",
    instruction="Refactor code based on review feedback."
)

# Create sequential pipeline
pipeline = SequentialAgent(
    name="code_pipeline",
    description="Code writing, review, and refactoring pipeline",
    sub_agents=[code_writer, code_reviewer, code_refactorer]
)
```

**Use When**:
- Steps must execute in fixed order
- Each step depends on previous output
- Deterministic workflow required

### ParallelAgent

Execute multiple agents simultaneously:

```python
from google.adk.agents import ParallelAgent

# Parallel research agents
tech_researcher = LlmAgent(name="tech_researcher", ...)
market_researcher = LlmAgent(name="market_researcher", ...)
competitor_researcher = LlmAgent(name="competitor_researcher", ...)

parallel_research = ParallelAgent(
    name="parallel_research",
    description="Concurrent market research",
    sub_agents=[tech_researcher, market_researcher, competitor_researcher]
)
```

**Use When**:
- Tasks are independent
- Reduce latency through concurrency
- Gather diverse perspectives simultaneously

### LoopAgent

Repeat until termination condition:

```python
from google.adk.agents import LoopAgent

iterative_improver = LlmAgent(
    name="improver",
    model="gemini-2.5-flash",
    instruction="""
    Review and improve the draft.
    Output 'DONE' when quality meets standards.
    """
)

loop_agent = LoopAgent(
    name="improvement_loop",
    description="Iteratively improve until quality threshold",
    sub_agent=iterative_improver,
    max_iterations=5,
    termination_phrase="DONE"
)
```

**Use When**:
- Iterative refinement needed
- Quality threshold must be reached
- Feedback loops required

### Nested Workflows

Combine workflow patterns:

```python
# Parallel phase followed by sequential synthesis
research_phase = ParallelAgent(
    name="research_phase",
    sub_agents=[researcher_1, researcher_2, researcher_3]
)

synthesis_phase = SequentialAgent(
    name="synthesis_phase",
    sub_agents=[aggregator, formatter, validator]
)

full_pipeline = SequentialAgent(
    name="research_pipeline",
    sub_agents=[research_phase, synthesis_phase]
)
```

## LLM-Driven Agent Transfer

The LLM decides when to delegate to sub-agents:

```python
# Root orchestrator with sub-agents
orchestrator = LlmAgent(
    name="customer_service",
    model="gemini-2.5-flash",
    instruction="""
    Route customer requests to appropriate specialists:
    - Billing issues → billing_agent
    - Technical problems → support_agent
    - Sales inquiries → sales_agent
    """,
    sub_agents=[billing_agent, support_agent, sales_agent]
)
```

**How It Works**:
1. LLM analyzes user request
2. LLM emits `AgentTransfer` event with target agent name
3. Control transfers completely to target agent
4. Target agent handles subsequent interactions

**Key Difference from AgentTool**: Transfer gives full control to target agent.

## Agent as Tool Pattern

Use agents as tools while maintaining orchestrator control:

```python
from google.adk.tools import AgentTool

# Wrap agents as tools
billing_tool = AgentTool(billing_agent)
support_tool = AgentTool(support_agent)
sales_tool = AgentTool(sales_agent)

coordinator = Agent(
    name="coordinator",
    model="gemini-2.5-flash",
    instruction="""
    Help customers by using specialist agents as tools.
    You can call multiple specialists and synthesize their responses.
    """,
    tools=[billing_tool, support_tool, sales_tool]
)
```

**Key Difference from Transfer**: Orchestrator stays in control, can combine multiple agents.

## Hierarchical Multi-Agent Systems

Build deep agent hierarchies:

```python
# Level 3: Specialized workers
data_collector = LlmAgent(name="data_collector", ...)
data_analyzer = LlmAgent(name="data_analyzer", ...)
report_writer = LlmAgent(name="report_writer", ...)

# Level 2: Department coordinators
research_dept = SequentialAgent(
    name="research_dept",
    sub_agents=[data_collector, data_analyzer]
)

marketing_dept = ParallelAgent(
    name="marketing_dept",
    sub_agents=[content_agent, social_agent, email_agent]
)

# Level 1: CEO orchestrator
ceo = LlmAgent(
    name="ceo",
    model="gemini-2.5-pro",  # More capable model for complex decisions
    instruction="Coordinate departments to achieve business goals.",
    sub_agents=[research_dept, marketing_dept, sales_dept]
)
```

## A2A Protocol (Agent-to-Agent)

Cross-organization agent communication (v0.3 - July 2025):

```python
from google.adk.agents import A2AAgent

# Connect to external A2A agent
salesforce_agent = A2AAgent(
    name="salesforce_crm",
    endpoint="https://partner.salesforce.com/agent/v1",
    authentication={
        "type": "oauth2",
        "client_id": os.environ["SF_CLIENT_ID"],
        "client_secret": os.environ["SF_CLIENT_SECRET"]
    }
)

# Local agent can delegate to remote A2A agent
local_coordinator = Agent(
    name="coordinator",
    model="gemini-2.5-flash",
    sub_agents=[salesforce_agent, local_support_agent]
)
```

**A2A Protocol Features**:
- Open standard (150+ organizations supporting)
- HTTP/gRPC transport
- JSON-RPC + Server-Sent Events
- Framework-agnostic (ADK, LangGraph, CrewAI compatible)

## Common Orchestration Patterns

### 1. Research Pipeline
```python
SequentialAgent(
    sub_agents=[
        ParallelAgent(sub_agents=[research_agents...]),  # Parallel research
        LlmAgent("synthesizer", ...),                    # Combine findings
        LlmAgent("fact_checker", ...),                   # Verify claims
        LlmAgent("formatter", ...)                       # Final output
    ]
)
```

### 2. Customer Service Routing
```python
LlmAgent(
    name="router",
    instruction="Route to appropriate specialist.",
    sub_agents=[billing, support, sales, escalation]
)
```

### 3. Quality Assurance Loop
```python
SequentialAgent(
    sub_agents=[
        LlmAgent("creator", ...),
        LoopAgent(
            sub_agent=LlmAgent("reviewer_improver", ...),
            max_iterations=3
        ),
        LlmAgent("finalizer", ...)
    ]
)
```

### 4. Fan-Out/Fan-In
```python
def fan_out_fan_in():
    parallel = ParallelAgent(
        sub_agents=[analyst_1, analyst_2, analyst_3]
    )

    aggregator = LlmAgent(
        name="aggregator",
        instruction="Synthesize all analyst reports into single summary."
    )

    return SequentialAgent(sub_agents=[parallel, aggregator])
```

## SoundSage Multi-Agent Example

For the ss-agent, consider this orchestration:

```python
# Specialized mixing agents
eq_specialist = LlmAgent(
    name="eq_specialist",
    instruction="Expert in frequency shaping and EQ decisions.",
    tools=[effects_tool]
)

dynamics_specialist = LlmAgent(
    name="dynamics_specialist",
    instruction="Expert in compression, limiting, transient shaping.",
    tools=[effects_tool]
)

spatial_specialist = LlmAgent(
    name="spatial_specialist",
    instruction="Expert in panning, stereo width, reverb, delay.",
    tools=[sends_tool, tracks_tool]
)

# Main orchestrator
soundsage_orchestrator = LlmAgent(
    name="soundsage_orchestrator",
    model="gemini-2.5-flash",
    instruction="""
    Analyze mixing requests and delegate to specialists:
    - EQ/frequency issues → eq_specialist
    - Dynamics/punch/loudness → dynamics_specialist
    - Space/width/depth → spatial_specialist

    Synthesize their recommendations into a cohesive action plan.
    """,
    sub_agents=[eq_specialist, dynamics_specialist, spatial_specialist]
)
```

## Best Practices

1. **Match pattern to problem**:
   - Sequential: Pipelines with dependencies
   - Parallel: Independent tasks
   - Loop: Iterative refinement
   - Hierarchical: Complex organizations

2. **Use appropriate model tiers**:
   - Workers: gemini-2.5-flash (fast, cheap)
   - Orchestrators: gemini-2.5-pro (better reasoning)

3. **Clear agent descriptions**: Essential for LLM-driven routing

4. **Limit hierarchy depth**: 2-3 levels typically sufficient

5. **Test each agent independently** before combining

6. **Monitor token usage**: Multi-agent systems can be expensive
