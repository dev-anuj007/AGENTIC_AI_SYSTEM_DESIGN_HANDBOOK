# Supervisor Pattern

## Problem

In a multi-agent system, different agents may specialize in different capabilities.

The system needs a component that decides:

* Which agent should work next
* What task it should receive
* Whether the result is good enough
* Whether another agent should be called
* When the overall task is complete

This is the **Supervisor Pattern**.

---

## Core Idea

A central supervisor coordinates multiple specialized agents.

```text
                    User Goal
                       │
                       ▼
                  Supervisor
                 /     |      \
                ▼      ▼       ▼
          Research   Coding     QA
             Agent    Agent    Agent
                \      |       /
                 ▼     ▼      ▼
                  Supervisor
                       │
              Evaluate Result
                       │
              ┌────────┴────────┐
              │                 │
           Continue            Done
              │                 │
              ▼                 ▼
        Delegate again       Final Result
```

The supervisor owns **coordination**, while specialized agents own **execution**.

---

## Example: AI Software Engineering Team

Suppose the user asks:

> "Add OAuth authentication to our application."

The supervisor may dynamically coordinate:

```text
Supervisor
    │
    ├── Research Agent
    │     → Understand existing authentication architecture
    │
    ├── Coding Agent
    │     → Implement OAuth integration
    │
    └── QA Agent
          → Run tests and security checks
```

If QA reports a failure:

```text
QA → "Token validation is failing"

Supervisor
    │
    └── Coding Agent
          → Fix token validation
                │
                ▼
              QA again
```

The supervisor therefore doesn't simply execute:

```text
Research → Coding → QA
```

It can **inspect results and decide what happens next**.

---

## Supervisor Responsibilities

The supervisor typically manages:

### 1. Task Decomposition

Break the overall goal into specialized tasks.

### 2. Agent Selection

Choose the appropriate agent based on its capabilities.

### 3. Context Passing

Give each agent only the information it needs.

### 4. Result Evaluation

Inspect whether the agent completed its task successfully.

### 5. Re-delegation

Send work back to an agent when the result is incomplete or incorrect.

### 6. Termination

Decide when the overall objective has been satisfied.

---

## Agent Contracts

Agents should expose clear contracts rather than sharing arbitrary internal state.

```text
Agent
├── capabilities
├── input_schema
├── output_schema
├── permissions
└── failure_contract
```

For example:

```json
{
  "agent": "qa_agent",
  "input": {
    "repository": "...",
    "changes": "..."
  },
  "output": {
    "status": "failed",
    "issues": ["token validation failure"]
  }
}
```

Structured contracts make delegation, validation, retries, and observability much easier.

---

## State

The supervisor needs state such as:

```text
goal
tasks
agent_outputs
task_status
current_agent
retry_count
iteration_count
```

This state allows the supervisor to understand what has already happened and decide what should happen next.

---

## Failure Handling

An agent failure does not necessarily mean the entire workflow should fail.

The supervisor can choose:

```text
Agent Failure
     │
     ├── Retry
     ├── Reassign
     ├── Ask another agent
     ├── Escalate to human
     └── Terminate
```

The decision should be based on the failure type and system policy.

---

## When to Use

Use a Supervisor when:

* Multiple specialized agents are required
* Tasks may depend on previous agent results
* The execution path can change dynamically
* A central coordinator simplifies governance
* Agents need different tools or permissions

---

## When NOT to Use

Avoid it when:

* One agent can solve the problem
* The workflow is completely deterministic
* Agents can operate independently without coordination
* Coordination overhead is larger than the actual work

For independent tasks, **Fan-Out/Fan-In** may be simpler.

---

## Trade-offs

| Advantage                | Cost                                    |
| ------------------------ | --------------------------------------- |
| Centralized coordination | Supervisor can become a bottleneck      |
| Dynamic delegation       | More complex execution                  |
| Easier governance        | Supervisor requires significant context |
| Specialized agents       | More infrastructure                     |
| Easier failure recovery  | More state management                   |

A poorly designed supervisor can become a **god agent** that tries to understand and control everything.

Keep the supervisor focused on **coordination**, not domain execution.

---

## Production Considerations

A production supervisor should have:

* Maximum iterations
* Maximum execution time
* Token/cost budgets
* Agent-level authorization
* Structured agent contracts
* Retry limits
* Checkpointing
* Idempotent agent operations
* Full execution tracing
* Audit logs

The supervisor should also prevent an agent from calling capabilities it was not authorized to use.

---

## Mental Model

> **Centralize coordination, decentralize specialized execution.**

The supervisor decides **who should do what next**.

The specialized agents decide **how to perform their assigned work**.
