# Reflection Pattern

## 1. Problem

An LLM can produce an answer that is plausible but incomplete, incorrect, or poorly structured.

For tasks where quality matters, we can introduce a second stage that **reviews the generated result and identifies improvements**.

Example:

> Generate a security report → review it → improve it.

---

## 2. Core Idea

Reflection adds a feedback loop around an output:

```text id="ref01"
Generate
   ↓
Review
   ↓
Improve
   ↓
Final Result
```

The reviewer examines the current result against defined criteria and produces feedback.

The generator then uses that feedback to produce an improved result.

---

## 3. Example: AI Security Report

Suppose an agent generates a security assessment.

### Initial generation

```text id="ref02"
Research
   ↓
Generate Report
```

### Reflection

The reviewer checks:

```text id="ref03"
Accuracy
Completeness
Evidence
Clarity
Policy compliance
```

It might identify:

```text id="ref04"
Missing evidence for CVE claim
Risk severity not justified
Executive summary incomplete
```

The generator then incorporates the feedback:

```text id="ref05"
Generate
   ↓
Reflect
   ↓
Feedback
   ↓
Improve
   ↓
Final Report
```

---

## 4. Reflection vs Evaluation

These concepts are related but serve different purposes.

### Evaluation

Answers:

> "How good is this output?"

Example:

```text id="ref06"
Score = 0.82
```

### Reflection

Answers:

> "What is wrong and how can the output be improved?"

Example:

```text id="ref07"
Missing evidence
   ↓
Add supporting evidence
```

An evaluator can therefore be used inside a reflection loop.

---

## 5. Reflection Strategies

### Same Model

The same LLM generates and reviews the result.

```text id="ref08"
LLM
 ↓
Generate
 ↓
Review
 ↓
Improve
```

Simple, but the model may repeat the same mistakes.

### Separate Reviewer

A different model or agent performs the review.

```text id="ref09"
Generator
    ↓
Output
    ↓
Reviewer
    ↓
Feedback
    ↓
Generator
```

This can provide greater separation between generation and critique.

### Rule-Based Review

Some checks should remain deterministic:

```text id="ref10"
Output
 ↓
Schema validation
 ↓
Policy validation
 ↓
Reflection
```

For example, required fields, formatting, or authorization constraints should not depend solely on an LLM's judgment.

---

## 6. When to Use

Use reflection when:

* Output quality is more important than minimum latency.
* Errors can be detected by reviewing the result.
* The output can be improved using concrete feedback.
* The task benefits from critique or verification.

Typical examples:

```text id="ref11"
Code generation
Research reports
Document generation
Reasoning-heavy answers
Content generation
Data analysis
```

---

## 7. When Not to Use

Avoid reflection when:

### The task is simple

A second LLM call may add unnecessary cost.

### There is no meaningful evaluation criterion

The reviewer cannot provide useful feedback if "better" is undefined.

### Latency is critical

Reflection adds additional model calls.

### Deterministic validation is sufficient

For example:

```text id="ref12"
JSON schema validation
Type checking
Unit tests
```

should be performed directly rather than replaced with an LLM reviewer.

---

## 8. Reflection Loop

Reflection can be repeated:

```text id="ref13"
Generate
   ↓
Review
   ↓
Good?
 ├── Yes → Finish
 └── No
      ↓
    Improve
      ↓
    Review
```

Production systems should always impose limits:

```text id="ref14"
MAX_REFLECTIONS
MAX_TIME
MAX_COST
```

Otherwise the system can repeatedly modify an already acceptable result.

---

## 9. Production Considerations

### Define Evaluation Criteria

The reviewer needs explicit criteria:

```text id="ref15"
accuracy >= threshold
required_sections present
claims supported
format valid
```

### Avoid Self-Agreement

A model reviewing its own output may fail to detect its original mistake.

For important tasks, consider:

* Separate evaluator
* Deterministic validation
* External evidence
* Multiple evaluators

### Track Improvement

Measure whether reflection actually improves results:

```text id="ref16"
Initial score: 0.72
After reflection: 0.88
```

If reflection consistently provides little improvement, it may not justify its cost.

---

## 10. Trade-offs

| Aspect       | Reflection                         |
| ------------ | ---------------------------------- |
| Main purpose | Improve an existing result         |
| Execution    | Generate → Review → Improve        |
| Quality      | Potentially higher                 |
| Latency      | Higher                             |
| Cost         | Higher                             |
| Complexity   | Medium                             |
| Main risk    | Repeated or ineffective refinement |

The fundamental trade-off is:

> **Higher potential quality in exchange for additional model calls, latency, and cost.**

---

## 11. Mental Model

```text id="ref17"
Generate
   ↓
Critique
   ↓
Improve
   ↓
Validate
   ↓
Finish
```

> **Reflection is a feedback mechanism that uses critique to improve an existing result.**

It should be used when the expected quality improvement justifies the additional execution cost.
