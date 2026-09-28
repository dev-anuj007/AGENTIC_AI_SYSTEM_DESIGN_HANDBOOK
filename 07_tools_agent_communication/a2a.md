# Agent-to-Agent (A2A)

## Problem

Tool calling allows an agent to interact with capabilities.

But sometimes an agent needs to collaborate with **another autonomous agent** rather than simply invoke a function.

The second agent may have its own reasoning, tools, state, and execution logic.

This creates an **Agent-to-Agent (A2A)** communication problem.

---

## Core Idea

Agents communicate through a defined protocol instead of sharing internal implementation details.

```text
Agent A
  │
  │ Task / Message
  ▼
Agent B
  │
  │ Result / Status
  ▼
Agent A
```

Each agent remains independently responsible for its own execution.

---

## Example: Software Development Platform

Suppose a development platform has separate agents:

```text
Coding Agent
     │
     │ "Review this implementation"
     ▼
Code Review Agent
     │
     ├── analyzes code
     ├── runs checks
     └── identifies issues
     │
     ▼
Review Result
     │
     ▼
Coding Agent
```

The Coding Agent does not need to know how the Review Agent performs its analysis.

It only needs the review agent's interface and result.

---

## End-to-End Flow

A typical A2A interaction looks like this:

```text
1. User
   │
   │ "Implement OAuth authentication"
   ▼
2. Coding Agent
   │
   │ implements feature
   ▼
3. Coding Agent
   │
   │ A2A request:
   │ "Review this implementation"
   ▼
4. Code Review Agent
   │
   ├── validates request
   ├── loads repository context
   ├── analyzes code
   ├── runs security checks
   └── produces findings
   │
   ▼
5. A2A Response
   │
   │ { status: "completed", findings: [...] }
   ▼
6. Coding Agent
   │
   ├── no issues → continue
   │
   └── issues found → modify code
                         │
                         ▼
                     Review Agent
                         │
                         ▼
                    Final Approval
```

The important point is that the **Code Review Agent owns its own execution**.

The Coding Agent does not control its internal reasoning or tools.

---

## Failure and Retry Handling

A2A failures should be classified before deciding whether to retry.

```text
A2A Request
     │
     ▼
Agent B
     │
     ├── Success ───────────────→ Continue
     │
     ├── Timeout / Network Error
     │          │
     │          └── Retry with backoff
     │
     ├── Temporary Agent Failure
     │          │
     │          └── Retry / Reassign
     │
     ├── Invalid Request
     │          │
     │          └── Do NOT retry blindly
     │
     └── Task Failure
                │
                ├── Fix / Re-delegate
                └── Escalate
```

### Retry Policy

Only retry failures that are likely to be temporary.

For example:

```text
Attempt 1
   ↓
Timeout
   ↓
Wait
   ↓
Attempt 2
   ↓
Timeout
   ↓
Wait
   ↓
Attempt 3
   ↓
Fail / Escalate
```

Use:

* Maximum retry count
* Exponential backoff
* Request timeout
* Overall task deadline
* Cost/token limits

### Idempotency

Retries can accidentally execute the same operation multiple times.

For side-effecting requests, use an idempotency key:

```text
request_id = "review-abc-123"
```

Agent B can recognize a duplicate request and avoid repeating an operation unnecessarily.

### Failure Ownership

The receiving agent should return structured failure information:

```json
{
  "status": "failed",
  "error": {
    "type": "temporary",
    "code": "DEPENDENCY_TIMEOUT",
    "message": "Static analysis service timed out"
  }
}
```

This allows Agent A to distinguish:

```text
Temporary failure → retry

Invalid request → fix request

Task failure → reconsider/delegate

Permanent failure → escalate
```

Do not use retries as a substitute for handling incorrect agent decisions.

---

## Agent vs Tool

The key distinction is autonomy.

```text
Tool
→ Performs an operation

Agent
→ Receives a goal
→ Reasons about the goal
→ May use tools
→ Produces a result
```

For example:

```text
search_repository()
```

is a tool operation.

Whereas:

```text
"Review this repository for security issues"
```

can be delegated to an autonomous security agent.

---

## Communication Contract

Agent communication should use structured messages.

```json
{
  "task": "review_code",
  "context": {
    "repository": "payments-api"
  },
  "expected_output": "security_findings"
}
```

The response should also be structured:

```json
{
  "status": "completed",
  "findings": [
    {
      "severity": "high",
      "description": "..."
    }
  ]
}
```

A clear contract prevents agents from depending on each other's internal state.

---

## A2A vs Agent-as-Tool

| Agent as Tool               | Agent-to-Agent                 |
| --------------------------- | ------------------------------ |
| Capability invocation       | Agent collaboration            |
| Usually request → response  | Can support richer interaction |
| Caller owns the workflow    | Both agents remain autonomous  |
| Encapsulation of capability | Collaboration between agents   |
| Simpler control model       | More coordination complexity   |

The distinction is not merely the transport mechanism.

It is about whether the called component is treated as a **capability** or as an **autonomous collaborator**.

---

## When to Use

Use agent-to-agent communication when:

* Multiple autonomous agents need to collaborate
* Agents have distinct responsibilities
* One agent may need another agent's reasoning
* Agents can operate independently
* Cross-agent interoperability is required

---

## When NOT to Use

Avoid A2A when:

* A simple function/tool is sufficient
* A single agent can perform the task
* The interaction is deterministic
* Introducing another autonomous agent adds unnecessary complexity

---

## Production Considerations

Agent communication should include:

* Agent identity
* Authentication
* Authorization
* Message schemas
* Correlation IDs
* Timeouts
* Retry policies
* Idempotency
* Capability boundaries
* Audit logs
* Distributed tracing
* Delegation limits

Also prevent uncontrolled delegation:

```text
Agent A
  ↓
Agent B
  ↓
Agent C
  ↓
Agent D
  ↓
...
```

Use delegation depth, cost, and execution limits.

---

## Trade-offs

**Benefits**

* Independent agent ownership
* Reusable specialized agents
* Dynamic collaboration
* Supports distributed agent systems

**Costs**

* Network latency
* More complex failure handling
* Harder debugging
* More authorization boundaries
* Potential delegation loops
* Higher operational cost

---

## Mental Model

> **Tool calling:** "Perform this operation."

> **Agent-to-Agent:** "Collaborate with this other autonomous agent."

The architectural shift is from **calling a capability** to **communicating with another decision-making component**.
