# Loop Workflow

## 1. Problem

Some workflows cannot determine the final number of iterations in advance.

For example, an AI system may repeatedly evaluate a generated answer until it satisfies a quality requirement:

```text id="lp01"
Generate
   ↓
Evaluate
   ↓
Good enough?
 ├── Yes → Finish
 └── No  → Improve
              ↓
           Evaluate
```

This is a **Loop Workflow**.

---

## 2. Core Idea

A loop repeatedly executes a predefined set of steps until a termination condition is satisfied.

```text id="lp02"
Start
  ↓
Execute
  ↓
Condition
  ├── Continue → Execute again
  └── Stop     → Finish
```

Unlike a sequential workflow, the same steps may execute multiple times.

The loop itself is deterministic: the application defines **what repeats** and **when execution must stop**.

---

## 3. Example: AI Response Quality

Suppose an AI system must generate a customer response that passes a validation check.

```text id="lp03"
Customer Request
      ↓
Generate Response
      ↓
Evaluate Response
      │
      ├── Pass → Return Response
      │
      └── Fail → Improve Response
                         ↓
                    Evaluate Again
```

Possible evaluation criteria:

```text id="lp04"
accuracy
completeness
format
policy compliance
```

The system continues until:

```text id="lp05"
quality_score >= threshold
```

or a safety limit is reached.

---

## 4. Termination Conditions

Every production loop needs an explicit termination strategy.

### Success Condition

```text id="lp06"
quality >= 0.9
    ↓
Stop
```

### Maximum Iterations

```text id="lp07"
iteration >= 5
    ↓
Stop
```

### Timeout

```text id="lp08"
execution_time > 30s
    ↓
Stop
```

### Failure

```text id="lp09"
Repeated failure
    ↓
Fallback / Escalate
```

A robust system often combines them:

```text id="lp10"
Stop when:

quality >= threshold
        OR
iterations >= MAX_ITERATIONS
        OR
time >= TIMEOUT
```

---

## 5. Loop State

Each iteration needs access to the current state.

Example:

```text id="lp11"
state = {
    "response": "...",
    "feedback": "...",
    "iteration": 2
}
```

After each iteration:

```text id="lp12"
Generate
   ↓
Evaluate
   ↓
Update state
   ↓
Generate again
```

Keeping iteration state explicit makes the workflow easier to checkpoint, debug, and resume.

---

## 6. When to Use

Use a loop when:

* The same processing steps may need to execute repeatedly.
* The number of iterations depends on runtime results.
* Each iteration improves or transforms the previous result.
* A clear termination condition exists.

Typical examples:

```text id="lp13"
Generate → Evaluate → Improve
Retry → Validate
Search → Evaluate → Search again
Plan → Execute → Check → Re-plan
Batch processing with pagination
```

---

## 7. When Not to Use

Avoid loops when:

### The number of iterations is fixed

Use a sequential workflow:

```text id="lp14"
A → B → C → D
```

### The system needs to choose among many capabilities

Use a router.

### Independent work should execute concurrently

Use parallel/fan-out.

### The loop depends on open-ended reasoning

A full agent loop may be more appropriate than a rigid workflow loop.

---

## 8. Loop vs Agent Loop

This distinction is important.

### Workflow Loop

The repeated steps are predefined:

```text id="lp15"
Generate
   ↓
Evaluate
   ↓
Improve
   ↓
Repeat
```

The application controls the loop.

### Agent Loop

The model can dynamically decide what action to take:

```text id="lp16"
Goal
 ↓
Decide
 ↓
Act
 ↓
Observe
 ↓
Decide again
```

Therefore:

> **A workflow loop repeats a known process; an agent loop dynamically chooses actions within a controlled runtime.**

---

## 9. Production Considerations

Loops can create runaway execution, especially when LLMs are involved.

Always define:

```text id="lp17"
MAX_ITERATIONS
MAX_EXECUTION_TIME
MAX_TOKEN_BUDGET
MAX_COST
```

Also consider:

* Idempotency
* Checkpointing
* Retry behavior
* Observability
* Stagnation detection

For example, if five iterations produce essentially the same result:

```text id="lp18"
Iteration 1 → score 0.72
Iteration 2 → score 0.73
Iteration 3 → score 0.73
Iteration 4 → score 0.73
```

continuing indefinitely provides little value.

The system should detect lack of progress and terminate or escalate.

---

## 10. Trade-offs

| Aspect       | Loop                        |
| ------------ | --------------------------- |
| Execution    | Repeated                    |
| Iterations   | Runtime-dependent           |
| Termination  | Explicit condition required |
| Flexibility  | Medium–High                 |
| Complexity   | Medium                      |
| Main benefit | Handles iterative processes |
| Main risk    | Runaway execution           |

The fundamental trade-off is:

> **Loops provide iterative flexibility but require strong termination and resource controls.**

---

## 11. Mental Model

```text id="lp19"
        ┌──────────────┐
        │    Execute   │
        └──────┬───────┘
               ↓
           Condition
          /         \
      Continue       Stop
         │             │
         └─────────────┘
```

> **A loop repeats a predefined process until a termination condition is reached.**

For agentic systems, the key architectural decision is whether you need a **controlled workflow loop** or a **dynamic agent loop**.

---

## 12. Chapter 2 Summary

The six workflow patterns now cover the primary deterministic orchestration structures:

| Pattern        | Core Problem                    |
| -------------- | ------------------------------- |
| Sequential     | Known ordered steps             |
| Parallel       | Independent work                |
| Router         | Select a predefined path        |
| Fan-Out/Fan-In | Split and combine work          |
| Conditional    | Execute based on a condition    |
| Loop           | Repeat until a condition is met |

These patterns can be composed:

```text id="lp20"
Router
  ↓
Sequential
  ↓
Fan-Out
  ↓
Parallel Processing
  ↓
Fan-In
  ↓
Conditional
  ↓
Loop
```

The important design principle is:

> **Start with deterministic workflow patterns and introduce agent autonomy only where the execution path genuinely requires dynamic reasoning.**
