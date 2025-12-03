---
description: Systematic orchestration framework for managing complex multi-task projects through agentic execution. Use when working on large features, refactoring efforts, or any task requiring coordinated parallel work.
---

# Agentic Orchestration Framework

This skill provides a systematic approach to breaking down complex tasks into parallel workstreams that can be executed by specialized agents.

## When to Use This Skill

- Large feature implementations requiring multiple components
- Codebase-wide refactoring or migrations
- Complex debugging requiring investigation across multiple systems
- Any task that can benefit from parallel execution

## Core Principles

### 1. Task Decomposition
Break complex tasks into independent, parallelizable units:
- Identify natural boundaries (files, modules, features)
- Ensure minimal dependencies between parallel tasks
- Define clear success criteria for each unit

### 2. Agent Specialization
Match tasks to the right agent type:
- **Explore agents**: Codebase investigation, finding patterns, understanding architecture
- **Code architects**: Designing solutions, planning implementations
- **Code reviewers**: Quality assurance, finding bugs
- **General-purpose agents**: Complex multi-step tasks

### 3. Parallel Execution
Maximize efficiency through concurrent work:
- Launch independent agents simultaneously
- Collect and synthesize results
- Handle dependencies by sequencing dependent tasks

## Orchestration Pattern

```
1. ANALYZE: Understand the full scope of the task
2. DECOMPOSE: Break into independent work units
3. ASSIGN: Match units to appropriate agent types
4. EXECUTE: Launch agents in parallel where possible
5. SYNTHESIZE: Combine results and resolve conflicts
6. VERIFY: Ensure all units completed successfully
```

## Example Usage

For a feature like "Add user authentication":

1. **Parallel Investigation Phase**
   - Agent 1: Explore existing auth patterns in codebase
   - Agent 2: Research auth libraries compatible with stack
   - Agent 3: Analyze database schema for user storage

2. **Architecture Phase** (after investigation)
   - Architect agent: Design auth system based on findings

3. **Parallel Implementation Phase**
   - Agent 1: Implement auth middleware
   - Agent 2: Create user model and migrations
   - Agent 3: Build login/signup UI components

4. **Review Phase**
   - Review agent: Check implementation quality

## Best Practices

- Always start with exploration to understand context
- Use TodoWrite to track orchestrated tasks
- Prefer smaller, focused agents over monolithic ones
- Synthesize agent outputs before presenting to user
- Handle failures gracefully with retry or alternative approaches
