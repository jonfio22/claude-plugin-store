---
name: tune-prompt
description: Analyze and optimize an ADK agent's system prompt for better performance
allowed_tools: [Read, Write, Edit, Glob, Grep]
---

# Tune ADK Agent Prompt

You are analyzing and optimizing a Google ADK agent's system prompt. Reference the sophisticated patterns from `/Users/fiorante/Documents/sound-sage-finalboss/ss-agent/prompts/`.

## Workflow

1. **Read Current Prompt** - Load the agent's system prompt
2. **Analyze Structure** - Check against best practices
3. **Identify Issues** - Find gaps, redundancy, ambiguity
4. **Propose Improvements** - Suggest specific changes
5. **Implement Changes** - Apply optimizations

## Prompt Analysis Checklist

### Structure (Does it have?)
- [ ] Clear role/identity definition
- [ ] Context/state documentation
- [ ] Tool reference section
- [ ] Decision rules
- [ ] Communication style guidelines
- [ ] Example interactions
- [ ] Error handling guidance

### Quality Metrics
- [ ] Token efficiency (no redundancy)
- [ ] Specificity (concrete, not vague)
- [ ] Actionability (clear instructions)
- [ ] Examples (realistic, diverse)
- [ ] Mode support (if multi-mode)

## Common Issues to Fix

### 1. Vague Instructions
```
# Before (vague)
"Be helpful and keep responses short"

# After (specific)
"Max 2 sentences in intern mode. Use studio shorthand (dB, EQ, comp)."
```

### 2. Missing Tool Documentation
```
# Before (missing)
"You have tools available"

# After (documented)
### tracks(action, selector, **params)
Control track properties. Selectors: "all", "selected", "group:Name"
Actions: volume, pan, mute, solo, arm, name, color
```

### 3. No Decision Framework
```
# Before (missing)
[No guidance on when to act vs. ask]

# After (clear rules)
## Decision Rules
1. Act immediately when request is clear and specific
2. Ask for clarification when target is ambiguous
3. Never call tools just to query state - you have it
4. Use selectors for batch operations - don't loop
```

### 4. Generic Examples
```
# Before (generic)
User: "Help me with audio"
Assistant: "I can help with that"

# After (realistic)
User: "The kick sounds flabby"
Context: Loose, unfocused low end
Action: EQ cut at 200-250Hz, tighten transient
Response: "Cut some low-mid mud and tightened the transient."
```

### 5. No Mode Differentiation
```
# Before (single mode)
[Same behavior for all contexts]

# After (mode-aware)
Based on mode ({mode}):

**Intern Mode**:
- Max 2 sentences
- Studio shorthand OK
- Action-focused, no explanations

**Mentor Mode**:
- 2-3 sentences with reasoning
- Explain the "why"
- Include listening tips
```

## Optimization Techniques

### Reduce Token Usage
1. Use tables instead of prose lists
2. Combine similar instructions
3. Remove redundant phrases
4. Use templates over repetition

### Improve LLM Understanding
1. Structure with headers
2. Use consistent formatting
3. Provide concrete examples
4. Define ambiguous terms

### Add Domain Vocabulary
```
## Vibe Translation

| Vibe | Technical Translation |
|------|----------------------|
| "warmer" | +2-4dB at 200-400Hz, tape saturation |
| "punchy" | Fast attack, boost 80-100Hz |
| "muddy" | Cut 200-400Hz |
| "harsh" | Cut 2-5kHz |
```

## Output Format

Provide:

1. **Current Prompt Analysis**
   - Structure score (X/7 sections)
   - Token count estimate
   - Issues identified

2. **Proposed Changes**
   - Before/after for each change
   - Rationale for change
   - Priority (high/medium/low)

3. **Optimized Prompt**
   - Full revised prompt
   - Token count comparison
   - Testing recommendations

## Example Optimization

**Before** (vague, unstructured):
```
You are an AI assistant that helps with audio. Be helpful and concise.
You have tools for mixing. Use them when needed.
```

**After** (structured, specific):
```
# Audio Mixing Assistant

You are an expert audio mixing assistant. Translate subjective descriptions
("warmer", "punchier") into precise mixing actions.

## Tools Reference

### effects(action, track_id, preset=None)
Apply effects: eq, compressor, saturation, limiter
Presets: punchy, gentle, warm_tape, transparent

### tracks(action, selector, value)
Adjust: volume, pan, mute, solo
Selectors: "all", "selected", "group:Drums"

## Decision Rules
1. Act when request is clear → apply effect/adjustment
2. Ambiguous target → ask "Which track/element?"
3. Vibe description → translate to technical params

## Communication
- Max 2 sentences
- State what you did, not what you're going to do
- Use dB, Hz, ms values

## Examples

User: "Kick is flabby"
Action: EQ cut 200-250Hz, tighten transient
Response: "Cut low-mid mud and tightened attack."
```
