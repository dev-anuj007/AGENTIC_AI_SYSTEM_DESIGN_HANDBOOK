# Conditional Workflow

## 1. Problem

Some workflows should not execute every step for every request.

A step may depend on a runtime condition:

```text id="cnd01"
Request
   ↓
Validate
   ↓
Condition
  ├── Yes → Process
  └── No  → Skip
```

Unlike a router, which primarily selects **which capability handles a request**, a conditional controls **whether a particular step or branch should execute**.

---

## 2. Core Idea

A conditional evaluates a known condition and follows the corresponding path.

```text id="cnd02"
Condition
   │
   ├── True  → Path A
   │
   └── False → Path B
```

The condition can be based on:

* Input data
* Previous step output
* Business rules
* System state
* Risk level
* Validation results

The possible paths are defined by the application.

---

## 3. Example: Customer Refund

Consider a support system processing refund requests.

Business rule:

```text id="cnd03"
Refund amount > ₹10,000
        ↓
Human approval required
```

Workflow:

```text id="cnd04"
Refund Request
      ↓
Validate Request
      ↓
Check Amount
      │
      ├── ≤ ₹10,000 ──→ Process Refund
      │
      └── > ₹10,000 ──→ Human Approval
                              ↓
                         Process Refund
```

The workflow is still deterministic. The runtime data determines which predefined path executes.

---

## 4. Conditional vs Router

They can look similar, but their intent differs.

| Pattern     | Question                                               |
| ----------- | ------------------------------------------------------ |
| Conditional | Should this step/branch execute?                       |
| Router      | Which predefined capability should handle the request? |

Example:

```text
Conditional:
amount > 10000?
    → approval
```

```text
Router:
request type?
    → billing
    → technical
    → sales
```

A router can contain conditionals internally.

---

## 5. Nested Conditions

Conditions can be composed when business rules require multiple decisions.

```text id="cnd05"
Request
   ↓
Authenticated?
   ├── No  → Reject
   └── Yes
         ↓
Amount > ₹10,000?
   ├── Yes → Approval
   └── No  → Process
```

Avoid deeply nested conditions when possible. If the decision logic becomes difficult to understand or maintain, extract it into a dedicated policy/rules component.

---

## 6. Conditional With LLM Output

An LLM can provide information used by a conditional, but the application should ideally own the actual business rule.

Example:

```text id="cnd06"
LLM
 ↓
Extract refund amount
 ↓
Application condition
 ↓
amount > 10,000?
```

Prefer:

```python
if refund_amount > 10_000:
    require_approval()
```

over asking the LLM:

> "Should this refund require approval?"

For deterministic business rules, keep the final decision deterministic.

---

## 7. When to Use

Use conditional workflows when:

* A step depends on runtime state.
* Business rules determine whether processing continues.
* Optional processing is required.
* Validation determines the next step.
* Approval or escalation depends on a threshold.

Typical examples:

```text
Validation
Approval
Fraud checks
Feature flags
Risk thresholds
Optional enrichment
Fallback decisions
```

---

## 8. When Not to Use

Avoid large conditional chains when:

* There are many independent capabilities → consider a router.
* Multiple branches should execute → use parallel/fan-out.
* The system needs repeated execution → use a loop.
* The decision requires open-ended reasoning → consider an agent.

---

## 9. Production Considerations

Important concerns include:

```text
Deterministic rules
Rule versioning
Auditability
Observability
Boundary testing
Default/fallback behavior
```

For important decisions, record:

```text
condition
input values
rule version
selected branch
timestamp
```

Example:

```text
amount = 15,000
rule = refund_v3
condition = amount > 10,000
result = approval_required
```

This makes business decisions explainable and auditable.

---

## 10. Trade-offs

| Aspect         | Conditional                       |
| -------------- | --------------------------------- |
| Decision       | Runtime condition                 |
| Paths          | Predefined                        |
| Predictability | High                              |
| Complexity     | Low initially                     |
| Best for       | Business rules and optional paths |
| Main risk      | Complex branching logic           |

The main trade-off is:

> **Simple conditionals make workflows flexible; too many conditions can turn the workflow into difficult-to-maintain business logic.**

---

## 11. Mental Model

```text
Input
  ↓
Condition
  │
  ├── True  → Path A
  │
  └── False → Path B
```

> **A conditional evaluates a known rule and controls whether a predefined path executes.**

Keep deterministic business decisions in the workflow or policy layer, and use LLMs for tasks where probabilistic reasoning is actually required.
