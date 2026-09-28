# Trajectory Evaluation

## Problem

Evaluating only the final answer does not tell us whether the agent followed a good execution path.

An agent may produce a correct result while:

* Calling unnecessary tools
* Accessing excessive data
* Taking an inefficient path
* Violating preferred policies
* Using too many LLM calls

Trajectory evaluation checks **how the agent reached the result**.

---

## Core Idea

Evaluate the sequence of decisions and actions taken during execution.

```text id="j7m4qx"
Goal
 ↓
Agent Decision
 ↓
Tool Call
 ↓
Observation
 ↓
Agent Decision
 ↓
Tool Call
 ↓
Final Result
        │
        ▼
Trajectory Evaluator
```

The evaluator examines the complete execution path rather than only the final output.

---

## Example: Security Investigation Agent

Suppose an agent investigates whether a company is vulnerable.

A valid trajectory might be:

```text id="p3k8vz"
1. Identify company domain
        ↓
2. Discover public assets
        ↓
3. Identify exposed services
        ↓
4. Query vulnerability database
        ↓
5. Validate evidence
        ↓
6. Generate finding
```

A problematic trajectory might:

```text
1. Search unrelated sources
2. Repeat the same query
3. Access unnecessary tools
4. Skip evidence validation
5. Generate conclusion
```

The final answer could still look reasonable, but the trajectory reveals inefficient or risky behavior.

---

## What to Evaluate

### Tool Selection

Did the agent choose appropriate tools?

```text
Query → Search Tool → Good
Query → Database Tool → Good
Query → Unrelated Admin Tool → Bad
```

### Tool Arguments

Were parameters valid and appropriately scoped?

### Ordering

Were dependencies respected?

```text
Authenticate
   ↓
Retrieve Data
   ↓
Modify Data
```

not:

```text
Modify Data
   ↓
Authenticate
```

### Efficiency

Did the agent perform unnecessary calls?

### Policy Compliance

Did the agent stay within its permissions?

### Recovery

Did the agent handle failures correctly?

### Termination

Did the agent stop when the task was complete?

---

## Structured Trajectory

Store the execution as structured events:

```json id="r5n8kx"
[
  {
    "step": 1,
    "action": "search_assets",
    "status": "success"
  },
  {
    "step": 2,
    "action": "query_vulnerabilities",
    "status": "success"
  },
  {
    "step": 3,
    "action": "validate_finding",
    "status": "success"
  }
]
```

This makes trajectory evaluation reproducible and measurable.

---

## Rule-Based Evaluation

Some trajectory properties can be checked deterministically.

```python id="m7q2vx"
def evaluate_trajectory(steps):
    tool_calls = len(steps)

    if tool_calls > 20:
        return "inefficient"

    for step in steps:
        if step["tool"] not in ALLOWED_TOOLS:
            return "policy_violation"

    return "valid"
```

These checks are useful for:

* Maximum tool calls
* Required steps
* Forbidden tools
* Ordering constraints
* Permission violations
* Timeout limits

---

## LLM-Based Evaluation

Some behaviors require semantic judgment.

For example:

> Was the agent's decision to perform another search justified by the previous observation?

An evaluator LLM can inspect:

```text
Goal
+
Previous observations
+
Agent decision
+
Tool result
```

and determine whether the next action was reasonable.

Use structured evaluator output:

```json id="q8m4zt"
{
  "decision": "pass",
  "issues": [],
  "efficiency": "good",
  "policy_compliance": "pass"
}
```

---

## Trajectory vs Final Output

They answer different questions.

| Final Output             | Trajectory               |
| ------------------------ | ------------------------ |
| Is the result good?      | Was the path good?       |
| Evaluates answer quality | Evaluates agent behavior |
| Correctness              | Tool selection           |
| Completeness             | Decision quality         |
| Groundedness             | Efficiency               |
| Format                   | Policy compliance        |

A production evaluation system may use both:

```text id="w2k7qa"
Agent Run
   │
   ├── Trajectory Evaluation
   │        ↓
   │    Was execution good?
   │
   └── Final Output Evaluation
            ↓
        Is result good?
```

---

## Production Considerations

Trajectory evaluation should capture enough information to reconstruct execution:

* Agent decisions
* Tool calls
* Tool arguments
* Tool results
* Retrieval operations
* Agent-to-agent calls
* Errors and retries
* Authorization decisions
* Execution time
* Token/cost information

Sensitive values should be redacted.

Use trace IDs to connect trajectory events across distributed services.

---

## When to Use

Trajectory evaluation is especially useful for:

* Complex agents
* Tool-heavy agents
* Multi-agent systems
* Long-running workflows
* Safety-sensitive systems
* Optimization of agent behavior
* Regression testing

---

## When NOT to Use

A separate trajectory evaluator may be unnecessary when:

* Execution is fully deterministic
* The workflow has very few steps
* There are no meaningful agent decisions
* Simple metrics already capture correctness

---

## Trade-offs

| Benefit                         | Cost                    |
| ------------------------------- | ----------------------- |
| Finds bad agent behavior        | More telemetry          |
| Improves efficiency             | Evaluation cost         |
| Detects policy violations       | More complex evaluation |
| Helps optimize agent strategies | Storage overhead        |

---

## Mental Model

> **Final output evaluation asks “Did we get the right result?” Trajectory evaluation asks “Did the agent take a good path to get there?”**

```text id="c4m9vx"
Trajectory → How did we get there?
Final Output → What did we produce?
```
