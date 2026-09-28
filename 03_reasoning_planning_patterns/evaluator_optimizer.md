# Evaluator-Optimizer Pattern

## 1. Problem

Some AI tasks require producing an output and then systematically checking whether it meets a quality standard.

A single generation step may not be reliable enough:

```text
Input
  ↓
Generator
  ↓
Output
```

The **Evaluator-Optimizer** pattern separates generation from evaluation and improvement.

---

## 2. Core Idea

Two components have distinct responsibilities:

**Generator / Optimizer**

* Produces or improves the result.

**Evaluator**

* Checks the result against defined criteria.
* Determines whether another iteration is required.

```text id="eo01"
              Input
                ↓
            Generator
                ↓
              Output
                ↓
            Evaluator
                │
          ┌─────┴─────┐
          ↓           ↓
        Pass         Fail
          ↓           ↓
        Final      Feedback
                        ↓
                    Optimizer
                        ↓
                    Evaluator
```

The important distinction from simple generation is the **explicit evaluation boundary**.

---

## 3. Example: Code Generation

Suppose an AI system must implement a feature.

```text id="eo02"
Requirement
    ↓
Code Generator
    ↓
Generated Code
    ↓
Evaluator
 ┌──┴───────────────┐
 ↓                  ↓
Tests Pass       Tests Fail
 ↓                  ↓
Finish           Feedback
                     ↓
                 Optimizer
                     ↓
                 New Code
                     ↓
                 Evaluator
```

The evaluator might check:

```text
Unit tests
Static analysis
Type checking
Security rules
Performance constraints
```

The optimizer uses the failures to produce a better implementation.

---

## 4. Evaluator vs Reflection

The patterns overlap, but the emphasis is different.

### Reflection

Focuses on:

```text
Generate
  ↓
Critique
  ↓
Improve
```

The critique is often qualitative.

### Evaluator-Optimizer

Focuses on a defined evaluation mechanism:

```text
Generate
  ↓
Evaluate against criteria
  ↓
Feedback
  ↓
Optimize
```

The evaluator can be:

* LLM-based
* Rule-based
* Test-based
* Metric-based
* Human

For engineering systems, deterministic evaluation should be preferred where possible.

---

## 5. Evaluation Criteria

The evaluator needs explicit criteria.

For the code example:

```text id="eo03"
correctness
test_pass_rate
type_safety
security
performance
```

For a research report:

```text id="eo04"
accuracy
evidence_coverage
completeness
format
```

Without measurable or clearly defined criteria, the optimizer has no reliable target.

---

## 6. Optimization Loop

The process can repeat:

```text id="eo05"
Generate
   ↓
Evaluate
   ↓
Score / Feedback
   ↓
Optimize
   ↓
Evaluate again
   ↓
...
   ↓
Pass / Maximum iterations
```

Production limits should include:

```text
MAX_ITERATIONS
MAX_EXECUTION_TIME
MAX_COST
```

The loop should stop when improvement is no longer worth the additional cost.

---

## 7. When to Use

Use Evaluator-Optimizer when:

* Output quality can be measured.
* Evaluation criteria can be explicitly defined.
* Feedback can be converted into an improved result.
* Additional iterations provide meaningful quality improvement.

Typical examples:

```text
Code generation
Research reports
SQL generation
Structured data extraction
Document generation
Agent trajectory optimization
```

---

## 8. When Not to Use

Avoid it when:

* The task is simple enough for direct generation.
* No reliable evaluation criteria exist.
* Evaluation costs more than the expected improvement.
* Deterministic validation alone solves the problem.
* Latency is more important than incremental quality.

For example, if the only requirement is valid JSON, use schema validation rather than an LLM evaluator.

---

## 9. Production Architecture

A production implementation can combine deterministic and LLM-based evaluation:

```text id="eo06"
                 Generator
                     ↓
              ┌──────┴──────┐
              ↓             ↓
         Unit Tests     LLM Evaluator
              │             │
              └──────┬──────┘
                     ↓
                  Decision
                 /        \
              Pass         Fail
               ↓             ↓
             Final       Optimizer
                             ↓
                         Generator
```

This is generally stronger than relying entirely on an LLM evaluator.

---

## 10. Observability

Track each optimization iteration:

```text id="eo07"
iteration
input
output
evaluation_score
feedback
failure_reason
latency
tokens
cost
termination_reason
```

A useful metric is **improvement per iteration**:

```text id="eo08"
Iteration 1 → 0.71
Iteration 2 → 0.84
Iteration 3 → 0.86
Iteration 4 → 0.86
```

If the score stops improving, further iterations may not provide value.

---

## 11. Trade-offs

| Aspect       | Evaluator-Optimizer                 |
| ------------ | ----------------------------------- |
| Main purpose | Iterative quality improvement       |
| Evaluation   | Explicit                            |
| Quality      | Potentially higher                  |
| Latency      | Higher                              |
| Cost         | Higher                              |
| Complexity   | Medium–High                         |
| Main benefit | Measurable refinement               |
| Main risk    | Expensive or ineffective iterations |

The fundamental trade-off is:

> **Use additional evaluation and optimization only when the expected quality improvement justifies the extra cost and latency.**

---

## 12. Mental Model

```text
Generate
   ↓
Evaluate
   ↓
Pass? ───── Yes → Finish
   │
   No
   ↓
Feedback
   ↓
Optimize
   ↓
Evaluate again
```

> **The evaluator defines what "good" means; the optimizer tries to make the output satisfy that definition.**

This makes Evaluator-Optimizer particularly useful when an AI system needs **measurable quality improvement rather than simply another generation attempt**.
