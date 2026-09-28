# Fallback Pattern

## Problem

An agentic system may depend on components that become unavailable or produce unusable results.

Instead of failing immediately, the system can switch to an **alternative execution path**.

This is the **Fallback Pattern**.

---

## Core Idea

```text id="m7q3vx"
Primary
   │
   ▼
Execute
   │
 ┌─┴──────────────┐
 │                │
Success          Failure
 │                │
 ▼                ▼
Continue       Fallback
                  │
                  ▼
              Alternative
                  │
                  ▼
                Result
```

A fallback is not simply another retry. It is a **different way of completing the task**.

---

## Example: Enterprise Research Agent

Suppose an agent needs vulnerability information.

The primary source is an internal security database.

```text id="k4n8pz"
Agent
  │
  ▼
Internal Security DB
  │
  ├── Available → Use result
  │
  └── Unavailable
          │
          ▼
     External Security API
          │
          ├── Available → Use result
          │
          └── Unavailable
                  │
                  ▼
              Report
            "Insufficient data"
```

The fallback source may provide less information, but it allows the system to continue safely.

---

## Fallback Types

### Service Fallback

Switch to another provider.

```text id="q3v6mx"
LLM Provider A
      ↓ failure
LLM Provider B
```

### Tool Fallback

Use another tool that provides a similar capability.

```text id="n8k4wp"
Primary Search
      ↓ failure
Alternative Search
```

### Model Fallback

Use a different model when the primary model is unavailable.

### Response Fallback

Return a safe partial result or explicitly state that required information is unavailable.

---

## Fallback vs Retry

The distinction is important:

```text id="c5m9rx"
Retry
→ Try the SAME operation again

Fallback
→ Try a DIFFERENT way to accomplish the goal
```

For example:

```text id="j2p7vn"
Payment API
   ↓ timeout
Retry Payment API
   ↓ still failing
Fallback → Payment Status API
```

---

## When to Use

Use fallbacks when:

* Multiple providers can perform the same capability
* Partial results are acceptable
* A dependency has known availability issues
* The system needs graceful degradation
* A safer alternative exists

---

## When NOT to Use

Avoid fallback when:

* The alternative is not trustworthy enough
* The operation requires an authoritative source
* Using the alternative could create incorrect or unsafe results
* The primary operation must succeed for correctness

Do not use fallback merely to hide failures.

---

## Production Considerations

Define explicitly:

* What failure triggers fallback
* Which fallback is allowed
* Data-quality differences
* Authorization requirements
* Timeout limits
* Cost limits
* Observability
* User-visible behavior

The system should know when **not** to fall back.

For example:

```text id="v9x4mk"
Primary unavailable
      ↓
Fallback available?
      │
 ┌────┴────┐
Yes        No
 │          │
 ▼          ▼
Fallback   Fail / Abstain
```

---

## Trade-offs

| Benefit                            | Cost                                       |
| ---------------------------------- | ------------------------------------------ |
| Higher availability                | More infrastructure                        |
| Graceful degradation               | Possible lower quality                     |
| Reduces dependency on one provider | More operational complexity                |
| Better user experience             | Different providers may behave differently |

---

## Mental Model

> **Retry asks: "Can I try the same operation again?"**

> **Fallback asks: "Can I accomplish the same goal another way?"**

A strong production system defines the fallback **before the failure happens**, rather than improvising after the primary dependency fails.
