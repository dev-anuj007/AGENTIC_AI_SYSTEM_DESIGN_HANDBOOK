# Agent vs Workflow

One of the first decisions in AI system design is:

> **Should this problem be implemented as a workflow, an agent, or a combination of both?**

The key distinction is **who determines the execution path**.

```text
Workflow
    → Developer defines the execution path

Agent
    → Model dynamically determines the execution path

Hybrid
    → Developer defines boundaries,
      Agent dynamically determines parts of the execution path
```

---

# 1. Workflow

A workflow follows a predefined sequence of steps.

```text
       Request
          │
          ▼
       Validate
          │
          ▼
       Retrieve
          │
          ▼
       Process
          │
          ▼
       Validate
          │
          ▼
       Response
```

The application determines:

* Which step runs next
* Which tools are called
* When execution stops
* How errors are handled

### Example

A document-processing system:

```text
Upload PDF
    ↓
Extract Text
    ↓
Extract Tables
    ↓
Generate Embeddings
    ↓
Store in Vector DB
    ↓
Complete
```

The execution path is known in advance.

---

# 2. Agent

An agent dynamically determines what action should happen next.

```text
                         Goal
                           │
                           ▼
                        Agent
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           Search          DB          API
              │            │            │
              └────────────┼────────────┘
                           ▼
                         Result
                           │
                           ▼
                         Agent
                           │
                     Continue / Stop
```

The developer provides the available capabilities, but the model determines which capability to use and in what order.

### Example

User:

> "Investigate why customer ACME's orders have been delayed and suggest what we should do."

The agent may decide:

```text
1. Get ACME customer information
2. Query recent orders
3. Check shipping status
4. Check inventory
5. Search support tickets
6. Analyze the findings
7. Recommend an action
```

The exact path depends on what the agent discovers.

---

# 3. Core Difference

The simplest distinction is:

```text
                    Who controls the path?
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
          Developer                   Model
              │                         │
              ▼                         ▼
          Workflow                    Agent
```

### Workflow

```text
A → B → C → D
```

### Agent

```text
A → ?
     │
     ├── B
     ├── C
     └── D

Then decide what happens next.
```

This difference is more important than whether an LLM is used.

A system can use an LLM extensively and still be a workflow.

---

# 4. Deterministic vs Dynamic Execution

### Workflow

Execution is mostly deterministic.

Given the same input and system state:

```text
Input
  ↓
A
  ↓
B
  ↓
C
```

The execution path is predictable.

### Agent

Execution is dynamic.

```text
Input
  ↓
Agent
  │
  ├── Tool A
  │
  ├── Tool C
  │
  └── Tool B
       │
       ▼
      Agent
       │
       └── Tool D
```

The path can change based on:

* User request
* Tool results
* Retrieved information
* Agent state
* Intermediate decisions

---

# 5. When to Use a Workflow

Use a workflow when:

### The process is well-defined

Example:

```text
Validate
 → Transform
 → Store
 → Notify
```

### The sequence matters

Example:

```text
Payment
  ↓
Update Order
  ↓
Send Confirmation
```

### Predictability is important

You know exactly which operations should occur.

### Strong control is required

You may need strict guarantees around:

* Security
* Compliance
* Cost
* Latency
* Data processing

### The task is repetitive

If the same steps execute for almost every request, introducing agent autonomy may add unnecessary complexity.

---

# 6. When to Use an Agent

An agent is useful when the execution path cannot easily be predefined.

### Open-ended problems

Example:

> "Research this company and identify potential security risks."

The agent may need to determine:

```text
What information should I collect?
Which sources should I search?
What should I investigate next?
Is the evidence sufficient?
```

### Dynamic tool selection

When the system has many possible capabilities:

```text
Search
Database
CRM
Email
Browser
Code execution
Internal APIs
```

and the appropriate sequence depends on the request.

### Iterative problem solving

Some tasks naturally involve:

```text
Attempt
 ↓
Observe
 ↓
Evaluate
 ↓
Adjust
 ↓
Try again
```

### Planning is required

When the system needs to decompose a goal into dynamically determined steps.

---

# 7. Hybrid Architecture

In production systems, the answer is often **both**.

Use deterministic workflows for predictable operations and agents for dynamic decisions.

```text
                    Request
                       │
                       ▼
                   Workflow
                       │
                       ▼
                    Router
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        Fixed Workflow         Agent
             │                   │
             ▼             ┌─────┼─────┐
          Database          ▼     ▼     ▼
                         Search  API   DB
             │                   │
             └─────────┬─────────┘
                       ▼
                    Validate
                       │
                       ▼
                    Response
```

This gives you:

```text
Workflow
  → Control and predictability

Agent
  → Flexibility and autonomy
```

A hybrid architecture does not mean the agent controls everything.

Instead:

```text
Developer
    │
    ▼
Define boundaries
    │
    ▼
Agent
    │
    ▼
Make dynamic decisions
    │
    ▼
Controlled execution
```

---

# 8. Example: Customer Support

Consider a support system.

## Pure Workflow

```text
User Question
      ↓
Classify
      ↓
Search FAQ
      ↓
Generate Answer
      ↓
Response
```

This works well when the possible paths are limited.

---

## Agent

```text
                   User Question
                         │
                         ▼
                       Agent
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      Search FAQ     Check Account   Search Tickets
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                       Agent
                         │
                    Need more data?
                    /          \
                  YES           NO
                   │             │
                   ▼             ▼
                Continue       Answer
```

The agent decides which information is necessary.

---

## Hybrid

A more controlled architecture might be:

```text
User
 │
 ▼
Workflow
 │
 ▼
Classify Request
 │
 ├── FAQ ──────────────→ Fixed RAG Workflow
 │
 ├── Billing ──────────→ Billing Agent
 │
 ├── Technical ────────→ Technical Agent
 │
 └── Sensitive ────────→ Human
```

This provides autonomy where useful while keeping sensitive or predictable operations controlled.

---

# 9. Defining Agent Autonomy Boundaries

Autonomy should not be treated as binary.

An agent can have different levels of autonomy:

```text
Low Autonomy                                      High Autonomy
     │                                                  │
     ▼                                                  ▼

Fixed       Router       Tool-Using      Planner       Autonomous
Workflow    + LLM       Agent            Agent          Agent
```

The important question is not:

> **"Is this an agent?"**

Instead ask:

> **"What decisions is the agent actually allowed to make?"**

---

## 9.1 Decision Boundary

Define which decisions the agent can make.

### Agent can decide

```text
✓ Which document to search
✓ Which API to query
✓ Which tool to use
✓ What information to retrieve
✓ Whether more information is required
```

### Agent should not decide

```text
✗ Authorization rules
✗ Business policies
✗ Compliance requirements
✗ System-wide security policies
```

The system should enforce these independently of the model.

---

## 9.2 Action Boundary

Define which actions the agent can execute.

```text
Agent
  │
  ├── Search documents      ✓
  ├── Read customer data    ✓
  ├── Create draft email    ✓
  ├── Send email            ⚠
  ├── Modify customer data  ⚠
  └── Delete production DB  ✗
```

The agent should not automatically have unrestricted access to every available tool.

---

## 9.3 Resource Boundary

Define which resources the agent can access.

```text
Agent
 │
 ├── Knowledge Base       ✓
 ├── Customer DB          ✓
 ├── Production DB        Read-only
 ├── Payment System       Restricted
 └── Admin System         ✗
```

This follows the principle of **least privilege**.

The agent receives only the data and capabilities required for its task.

---

## 9.4 Termination Boundary

The agent should not be allowed to run indefinitely.

The runtime should enforce limits such as:

```text
max_iterations
max_execution_time
max_token_budget
max_tool_calls
```

Example:

```text
Agent
  │
  ▼
Tool
  │
  ▼
Evaluate
  │
  ├── Continue
  │
  └── Stop
       │
       ├── Goal achieved
       ├── Max iterations reached
       ├── Timeout
       └── Budget exceeded
```

The agent can request another action, but the **runtime has the final authority to terminate execution**.

---

# 10. Example: Refund System

Consider an agent responsible for customer refunds.

### Poor boundary

```text
Customer
   ↓
Agent
   ↓
"I should refund $10,000"
   ↓
Payment API
```

The agent effectively controls the business policy.

---

### Controlled boundary

```text
Customer
   ↓
Agent
   ↓
Request Refund
   ↓
Policy Engine
   │
   ├── ≤ $100       → Automatically allowed
   ├── $100-$1,000  → Human approval
   └── > $1,000     → Not allowed
```

The agent can decide:

> "A refund is appropriate."

But the system decides:

> "Whether this agent is authorized to issue this refund."

This separation is critical.

---

# 11. Decision Framework

Use these questions during system design:

```text
                 Is the execution path known?
                       /              \
                     YES              NO
                      │                │
                      ▼                ▼
                  Workflow           Agent
```

If it is partially known:

```text
                 Is part of the process fixed?
                       /              \
                     YES              NO
                      │                │
                      ▼                ▼
                   Hybrid            Agent
```

Then consider:

```text
Does the system need dynamic tool selection?
        │
       YES → Agent capability

Does the system need planning?
        │
       YES → Agent capability

Does the system need iterative decisions?
        │
       YES → Agent capability

Does the system require strict predictable execution?
        │
       YES → Workflow

Does the system have high-risk operations?
        │
       YES → Controlled agent + approval/policy checks
```

---

# 12. Complexity Trade-off

Agentic systems provide flexibility, but that flexibility comes with additional complexity.

```text
                    Flexibility
                         ▲
                         │
                      Agent
                         │
                      Hybrid
                         │
                     Workflow
                         │
                         └──────────────►
                            Predictability
```

As autonomy increases, additional infrastructure is generally required:

```text
Agent
  ↓
State
  ↓
Memory
  ↓
Guardrails
  ↓
Observability
  ↓
Evaluation
  ↓
Reliability
```

Therefore:

> **Do not introduce agentic behavior simply because an LLM is involved.**

Use the minimum level of autonomy required by the problem.

---

# 13. Common Mistake

A common architecture mistake is:

```text
User
 ↓
LLM Agent
 ↓
Everything
```

where the agent is allowed to decide every operation.

For example:

```text
Agent decides:
- validation
- database writes
- payment
- notification
- retry
- authorization
- business rules
```

This can make the system difficult to:

* Debug
* Test
* Secure
* Predict
* Operate
* Control

A better architecture separates deterministic responsibilities from dynamic decisions.

```text
                 Agent
                   │
            Dynamic decisions
                   │
                   ▼
              Workflows
                   │
            Deterministic work
                   │
                   ▼
            External Systems
```

The principle is:

> **Let the agent decide within the boundary; let the system define the boundary.**

---

# 14. Summary

| Dimension       | Workflow           | Agent                        |
| --------------- | ------------------ | ---------------------------- |
| Execution path  | Predefined         | Dynamic                      |
| Decision maker  | Application        | Model                        |
| Predictability  | High               | Lower                        |
| Flexibility     | Lower              | High                         |
| Tool selection  | Usually predefined | Dynamic                      |
| Planning        | Usually predefined | Dynamic                      |
| Debugging       | Simpler            | More complex                 |
| Testing         | More deterministic | More difficult               |
| Control         | Strong             | Requires explicit boundaries |
| Best suited for | Known processes    | Open-ended tasks             |

The key mental model is:

```text
Workflow
    = Developer decides what happens next

Agent
    = Model decides what happens next

Hybrid
    = Developer defines boundaries,
      Agent handles dynamic decisions
      within those boundaries
```

The fundamental principle:

> **Agent autonomy should exist inside a controlled boundary, not outside the system's policies.**

In production systems, **hybrid architectures provide a practical balance between deterministic software engineering and dynamic agent behavior.**
