# Router Workflow

## 1. Problem

A sequential workflow follows a fixed path:

```text id="rtr01"
A → B → C
```

A router is useful when the system has **multiple predefined paths** and the appropriate path depends on the incoming request.

Example:

```text id="rtr02"
Request
   ↓
 Router
   ├── Billing
   ├── Technical
   ├── Sales
   └── Unknown
```

Only the selected route executes.

---

## 2. Core Idea

A router has four stages:

```text id="rtr03"
Input
  ↓
Classify / Evaluate
  ↓
Validate Route
  ↓
Dispatch
```

The possible routes are defined by the application.

The runtime decision determines **which route executes**, not what new route should be invented.

For example:

```text id="rtr04"
Allowed:
billing
technical
sales
account
unknown
```

---

## 3. Routing Strategies

The routing decision can use different signals.

### Rule-Based

```python
if request.type == "refund":
    route = "billing"
```

Best when routing criteria are deterministic.

### Classifier

```text id="rtr05"
Request
   ↓
Classifier
   ↓
billing
```

Useful when intent classification is difficult to express with simple rules.

### LLM Router

```text id="rtr06"
Request
   ↓
LLM
   ↓
{
    "route": "billing",
    "confidence": 0.96
}
```

Useful for natural-language requests with many possible intents.

### Metadata-Based

Routing can also use:

```text id="rtr07"
tenant
user_role
product
region
request_type
risk_level
```

In production, these signals can be combined.

---

## 4. Structured Routing

For LLM-based routing, use constrained structured output rather than free-form text.

```json id="rtr08"
{
  "route": "billing",
  "confidence": 0.96
}
```

Validate the route against an allowlist:

```python
if route not in ALLOWED_ROUTES:
    route = "unknown"
```

This makes routing:

* Testable
* Observable
* Validatable
* Easier to debug

---

## 5. Example: Customer Support

Suppose a support system has:

```text id="rtr09"
Billing
Technical
Account
Sales
```

Request:

> "I was charged twice for my subscription."

The router produces:

```text id="rtr10"
billing
   ↓
Billing Handler
```

Request:

> "The API is returning a 401."

becomes:

```text id="rtr11"
technical
   ↓
Technical Handler
```

The routing layer only determines ownership. The selected handler owns the actual business logic.

---

## 6. Unknown and Fallback Routes

Never assume every request belongs to a known category.

```text id="rtr12"
                 Router
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
    Billing     Technical    Unknown
```

The unknown path can:

* Ask for clarification
* Perform additional classification
* Send the request to a general handler
* Escalate to a human
* Reject unsupported requests

Forcing an ambiguous request into an arbitrary route can be worse than delaying execution for clarification.

---

## 7. Confidence-Based Routing

When classification is probabilistic, confidence can influence the next step:

```text id="rtr13"
High confidence
      ↓
Direct dispatch

Medium confidence
      ↓
Additional validation

Low confidence
      ↓
Clarification / Human
```

The threshold should depend on the consequence of an incorrect route.

A wrong FAQ route may be recoverable.

A wrong financial or administrative route may require stronger controls.

---

## 8. Router With Agents

A router can dispatch to specialized agents:

```text id="rtr14"
User
 ↓
Router
 ├── Research Agent
 ├── Coding Agent
 └── Sales Agent
```

The selected agent may then execute its own reasoning loop:

```text id="rtr15"
Router
  ↓
Research Agent
  ↓
Search
  ↓
Analyze
  ↓
Search again
  ↓
Finish
```

This separates two responsibilities:

```text
Router
→ Which capability should handle this?

Agent
→ What should happen during execution?
```

---

## 9. Router + Other Workflow Patterns

A route can lead to any other orchestration pattern.

### Router → Sequential

```text id="rtr16"
Request
  ↓
Router
  ↓
A → B → C
```

### Router → Parallel

```text id="rtr17"
Request
  ↓
Router
  ↓
┌────────┬────────┐
A        B
└────────┴────────┘
     ↓
   Merge
```

### Router → Agent

```text id="rtr18"
Request
  ↓
Router
  ↓
Agent Loop
```

Therefore, routing is often an **entry-point pattern**, while the selected capability can have its own internal architecture.

---

## 10. Hierarchical Routing

Large systems may require multiple routing levels:

```text id="rtr19"
                 Root Router
                     │
             ┌───────┴───────┐
             ↓               ↓
       Business Router   Technical Router
             │               │
        ┌────┼────┐      ┌───┼───┐
        ↓    ↓    ↓      ↓   ↓   ↓
      Sales Billing HR   API  DB Cloud
```

Use hierarchical routing when the capability space is large enough that one router becomes difficult to maintain.

Trade-off:

```text
Smaller routers
     ↓
Better separation

More routing layers
     ↓
More latency + complexity
```

---

## 11. Security

Routing is **not authorization**.

Incorrect:

```text
Router
  ↓
Admin Agent
```

The router selecting an admin capability does not mean the user is authorized to use it.

Prefer:

```text
Request
   ↓
Authentication
   ↓
Authorization
   ↓
Router
   ↓
Allowed Handler
```

Authorization should be enforced independently of an LLM routing decision.

This matters especially for:

* Administrative operations
* Financial operations
* Sensitive data
* Privileged tools

---

## 12. Failure Modes

### Misclassification

```text
Billing Request
      ↓
Technical Handler
```

### Invalid Route

```text
LLM → "financial_admin_v2"

Allowed:
billing
technical
sales
```

### Unknown Request

```text
Ambiguous Request
      ↓
Unknown
      ↓
Clarification
```

### Handler Unavailable

```text
Router
  ↓
Billing
  ↓
Unavailable
  ↓
Fallback / Retry / Escalation
```

### Routing Loop

```text
Router
 ↓
Agent A
 ↓
Router
 ↓
Agent A
 ↓
...
```

Prevent these with:

* Route allowlists
* Unknown/fallback handling
* Availability checks
* Explicit routing boundaries
* Maximum routing depth

---

## 13. Production Design

A production router can look like:

```text
Request
   ↓
AuthN / AuthZ
   ↓
Router
   ↓
Route Validation
   ↓
Handler / Agent
```

Useful telemetry:

```text
request_id
route
confidence
routing_latency
selected_handler
fallback
routing_error
execution_latency
```

For LLM routing, also track:

```text
model_version
prompt_version
classification_result
```

This allows routing accuracy and regressions to be evaluated over time.

---

## 14. When to Use

Use a router when:

* Multiple specialized capabilities exist.
* A request normally belongs to one primary capability.
* Routes can be explicitly defined.
* Different routes require different tools, policies, or data.
* Separation between domains is valuable.

Typical examples:

```text
Customer Support
Multi-Agent Platforms
Document Classification
Model Selection
Tool Selection
Enterprise Assistants
API Dispatch
```

---

## 15. When Not to Use

Avoid a router when:

### Only one capability exists

There is nothing to route.

### All branches should execute

Use parallel or fan-out/fan-in.

### The system requires continuous planning

Use an agentic loop or planning architecture.

### Routing rules are becoming extremely complex

Consider hierarchical routing or a more explicit workflow.

### Incorrect routing has high consequences

Add deterministic validation, authorization, or human approval.

---

## 16. Trade-offs

| Aspect         | Router                |
| -------------- | --------------------- |
| Routes         | Predefined            |
| Decision       | Runtime               |
| Flexibility    | Medium                |
| Predictability | Medium–High           |
| Complexity     | Low–Medium            |
| Main benefit   | Capability separation |
| Main risk      | Incorrect routing     |

The primary trade-off is:

```text
More specialized capabilities
        ↓
Better separation
        ↓
More routing complexity
```

---

## 17. Interview Example

### Question

> "You are building a customer-support platform with billing, technical, sales, and account-management agents. How would you architect it?"

Answer:

> "I would use a routing layer that classifies the request into a constrained set of supported capabilities and dispatches it to the corresponding handler. I would validate the route, provide an unknown/fallback path, and keep authorization separate from routing. For ambiguous or high-risk requests, I would require additional validation or human escalation."

Architecture:

```text
                     User
                       ↓
                   AuthN/AuthZ
                       ↓
                     Router
                       ↓
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
     Billing       Technical        Sales
      Agent          Agent          Agent
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                    Response
```

---

## 18. Mental Model

```text
Router
  ↓
Classify
  ↓
Validate
  ↓
Dispatch
```

> **A router selects a predefined capability. The selected capability owns the execution.**

The capability may itself be a sequential workflow, parallel workflow, or autonomous agent.

---

## 19. Chapter 2 Relationship

| Pattern        | Problem Solved               |
| -------------- | ---------------------------- |
| Sequential     | Known ordered steps          |
| Parallel       | Independent work             |
| Router         | Select one predefined path   |
| Fan-Out/Fan-In | Split and combine work       |
| Conditional    | Execute based on a condition |
| Loop           | Repeat execution             |

The important distinction is that these patterns describe **orchestration structure**. They can be composed rather than treated as mutually exclusive architectures.
