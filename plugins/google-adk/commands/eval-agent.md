---
name: eval-agent
description: Evaluate an ADK agent's performance with test cases, trajectories, and response quality
allowed_tools: [Read, Write, Edit, Glob, Bash, Grep]
---

# Evaluate ADK Agent

You are evaluating a Google ADK agent's performance. Reference the testing patterns from `/Users/fiorante/Documents/sound-sage-finalboss/ss-agent/tests/`.

## Gather Information

Ask the user for:
1. **Agent Path** - Path to the agent directory
2. **Evaluation Type**:
   - `quick` - Basic functionality tests
   - `comprehensive` - Full test suite
   - `benchmark` - Performance metrics
3. **Focus Areas** (optional):
   - Tool usage
   - Response quality
   - Mode behavior
   - Error handling

## Quick Evaluation

### 1. Run Existing Tests
```bash
cd {agent-path}
pytest tests/ -v --tb=short
```

### 2. ADK Web Evaluation
```bash
adk web --evaluate {agent-path}/
```

### 3. Manual Spot Checks
Test these scenarios:
- Basic greeting/capability question
- Simple tool invocation
- Multi-tool chain
- Ambiguous request (should clarify)
- Invalid input (should handle gracefully)
- Mode switching (if applicable)

## Comprehensive Evaluation

### Test Case File Format

Create `eval_cases.json`:
```json
{
  "test_cases": [
    {
      "id": "basic_001",
      "category": "basic",
      "input": "Set the drums to -6dB",
      "expected_tools": ["tracks"],
      "expected_params": {
        "action": "volume",
        "selector": "group:Drums"
      },
      "expected_response_contains": ["drum", "-6"],
      "mode": "intern"
    },
    {
      "id": "chain_001",
      "category": "chain",
      "input": "Make the vocals radio-ready",
      "expected_tools": ["effects", "effects", "effects", "sends"],
      "expected_response_contains": ["vocal", "chain"],
      "mode": "intern"
    },
    {
      "id": "clarify_001",
      "category": "clarification",
      "input": "Make it louder",
      "expected_tools": [],
      "expected_response_contains": ["which", "?"],
      "mode": "intern"
    },
    {
      "id": "error_001",
      "category": "error",
      "input": "Set track NonExistent to -3dB",
      "expected_tools": [],
      "expected_response_contains": ["not found", "don't see"],
      "mode": "intern"
    },
    {
      "id": "mentor_001",
      "category": "mode",
      "input": "Boost the vocals 2dB",
      "expected_tools": ["tracks"],
      "expected_response_contains": ["because", "helps", "listen"],
      "mode": "mentor"
    }
  ]
}
```

### Run Evaluation
```bash
adk eval --eval_set eval_cases.json {agent-path}/
```

## Performance Benchmark

Create `benchmark.py`:
```python
import asyncio
import time
import statistics
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from {package}.agent import root_agent

BENCHMARK_QUERIES = [
    "Set all drums to -3dB",
    "Apply punchy compression to the kick",
    "Make the mix warmer",
    "What's the current BPM?",
    "Undo the last change",
]

async def benchmark():
    session_service = InMemorySessionService()
    runner = Runner(agent=root_agent, session_service=session_service)

    results = {
        "latencies": [],
        "successes": 0,
        "failures": 0,
    }

    for query in BENCHMARK_QUERIES:
        for _ in range(5):  # 5 iterations each
            session = session_service.create_session(
                app_name="benchmark",
                user_id="bench"
            )

            start = time.time()
            try:
                result = await runner.run_async(query, session=session)
                results["latencies"].append(time.time() - start)
                results["successes"] += 1
            except Exception as e:
                results["failures"] += 1
                print(f"Failed: {query} - {e}")

    # Calculate metrics
    print("\n=== Benchmark Results ===")
    print(f"Total queries: {len(BENCHMARK_QUERIES) * 5}")
    print(f"Successes: {results['successes']}")
    print(f"Failures: {results['failures']}")
    print(f"Success rate: {results['successes'] / (results['successes'] + results['failures']) * 100:.1f}%")
    print(f"Avg latency: {statistics.mean(results['latencies']):.2f}s")
    print(f"P50 latency: {statistics.median(results['latencies']):.2f}s")
    print(f"P95 latency: {statistics.quantiles(results['latencies'], n=20)[18]:.2f}s")
    print(f"Max latency: {max(results['latencies']):.2f}s")

if __name__ == "__main__":
    asyncio.run(benchmark())
```

Run:
```bash
python benchmark.py
```

## Evaluation Dimensions

### 1. Trajectory Quality
- Did agent select correct tools?
- Was execution order optimal?
- Were unnecessary tools avoided?

### 2. Tool Usage
- Correct parameters?
- Appropriate selectors?
- Presets used when applicable?

### 3. Response Quality
- Correct and relevant?
- Appropriate length for mode?
- Professional tone?

### 4. Error Handling
- Graceful failure?
- Helpful error messages?
- Appropriate clarification requests?

## Report Format

Generate evaluation report:

```
=== Agent Evaluation Report ===
Agent: {agent_name}
Date: {date}
Test Cases: {total}

## Summary
- Pass Rate: {pass_count}/{total} ({percentage}%)
- Avg Latency: {avg_latency}s
- Tool Accuracy: {tool_accuracy}%
- Response Quality: {quality_score}/10

## By Category
| Category | Pass | Fail | Rate |
|----------|------|------|------|
| Basic    | X    | Y    | Z%   |
| Chain    | X    | Y    | Z%   |
| Clarify  | X    | Y    | Z%   |
| Error    | X    | Y    | Z%   |
| Mode     | X    | Y    | Z%   |

## Failed Cases
1. {case_id}: {failure_reason}
2. {case_id}: {failure_reason}

## Recommendations
1. {recommendation_1}
2. {recommendation_2}
3. {recommendation_3}
```

## Improvement Workflow

After evaluation:
1. Review failed cases
2. Identify root causes:
   - Prompt issue → `/tune-prompt`
   - Tool issue → `/new-tool` or fix existing
   - Configuration issue → adjust agent settings
3. Apply fixes
4. Re-run failing tests
5. Add regression tests
6. Run full suite to verify no regressions
