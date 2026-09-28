# Final Output Evaluation

## Problem

An agent can complete its workflow successfully and still produce a poor final answer.

For example:

```text
Tools succeeded
      ↓
Agent completed
      ↓
Final answer
      ↓
❌ Incorrect / incomplete / unsupported
```

Therefore, **workflow success is not the same as output quality**.

Final output evaluation checks whether the result satisfies the user's goal and required quality criteria.

---

## Core Idea

Evaluate the final result against explicit criteria before returning it.

```text id="q6v8mx"
Agent Execution
      │
      ▼
Final Output
      │
      ▼
Evaluator
      │
      ├── Good → Return
      │
      └── Poor → Improve / Retry / Escalate
```

The evaluator should judge the **result**, not simply whether the agent reached the final node.

---

## Example: AI Security Report

Suppose an agent generates a security assessment.

The evaluator checks:

```text id="k3p7qa"
Security Report
 │
 ├── Accuracy
 ├── Completeness
 ├── Evidence / Citations
 ├── Risk classification
 ├── Actionable recommendations
 └── Required format
```

If the report contains unsupported claims:

```text id="m8r2vx"
Generate Report
      │
      ▼
Evaluate
      │
      ├── Pass → Return
      │
      └── Fail
            │
            ▼
       Improve Report
            │
            ▼
         Evaluate
```

---

## Evaluation Criteria

Criteria should be defined before evaluation.

Common dimensions include:

### Correctness

Is the answer factually correct?

### Completeness

Did the agent address the important parts of the request?

### Relevance

Does the answer stay focused on the user's goal?

### Groundedness

Are claims supported by retrieved evidence when required?

### Format

Does the output satisfy the required structure or schema?

### Safety

Does the response avoid prohibited or unsafe actions?

---

## Rule-Based vs LLM Evaluation

### Rule-Based

Useful when correctness can be deterministically checked.

```python id="v9m2qx"
def evaluate_output(result):
    if not result.get("citations"):
        return "fail"

    if len(result["summary"]) < 100:
        return "fail"

    return "pass"
```

### LLM-Based

Useful for subjective or semantic criteria.

```text
Output
  ↓
Evaluator LLM
  ↓
{
  "score": ...,
  "issues": [...],
  "decision": "pass"
}
```

A production system often combines both.

---

## Structured Evaluation

Avoid relying on free-form evaluator responses.

Prefer:

```json id="n5q7rc"
{
  "decision": "fail",
  "criteria": {
    "correctness": "pass",
    "completeness": "fail",
    "groundedness": "pass"
  },
  "issues": [
    "Missing evidence for vulnerability severity"
  ]
}
```

This makes evaluation machine-actionable.

---

## Evaluation Loop

A simple quality-control loop:

```text id="r4k8vp"
Generate
   │
   ▼
Evaluate
   │
   ├── Pass ──→ Return
   │
   └── Fail
        │
        ▼
      Improve
        │
        ▼
      Evaluate
```

The loop must have bounded retries:

```text id="t7m3qa"
MAX_EVALUATION_ATTEMPTS
MAX_COST
MAX_TIME
```

Otherwise a poor output can cause an endless optimization loop.

---

## Final Output Evaluation vs Trajectory Evaluation

They evaluate different things.

| Final Output             | Trajectory                     |
| ------------------------ | ------------------------------ |
| Was the result good?     | Was the process good?          |
| Evaluates final answer   | Evaluates decisions/tool usage |
| Correctness/completeness | Tool selection/path            |
| User-facing quality      | Agent behavior                 |

An agent can have:

```text
Bad trajectory + Good result
```

or:

```text
Good trajectory + Bad result
```

Both are possible.

---

## Production Considerations

A production evaluator should consider:

* Explicit evaluation criteria
* Deterministic checks where possible
* Structured evaluator output
* Independent evaluator when appropriate
* Evaluation latency
* Evaluation cost
* Retry limits
* Versioned evaluation criteria
* Human review for high-impact results
* Evaluation metrics over time

Do not automatically treat an LLM evaluator as ground truth.

---

## When to Use

Final output evaluation is especially useful when:

* Quality is critical
* Outputs are complex
* Errors are expensive
* Generated reports require evidence
* Agents produce code
* Human review is expensive
* Outputs must follow strict schemas

---

## When NOT to Use

A separate evaluator may be unnecessary when:

* Output is deterministic
* A simple schema validator is sufficient
* The task is low-risk
* Evaluation costs more than the potential benefit

---

## Trade-offs

| Benefit                           | Cost                       |
| --------------------------------- | -------------------------- |
| Improves output quality           | Additional latency         |
| Catches failures                  | Additional model/tool cost |
| Enables automated quality control | Evaluator errors           |
| Supports iterative improvement    | More system complexity     |

---

## Mental Model

> **Successful execution means the agent finished. Evaluation determines whether what it produced is actually good enough to return.**

```text id="p8w3km"
Execute
   ↓
Generate Result
   ↓
Evaluate
   ↓
┌──────────────┐
│ Good enough? │
└──────┬───────┘
       │
   Yes │ No
       │
       ▼
     Return
       │
       └──→ Improve → Evaluate
```
