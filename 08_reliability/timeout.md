# Timeout Pattern

## Problem

A dependency may become slow or stop responding.

Without a timeout, an agent execution can remain blocked indefinitely:

```text id="q6m2vx"
Agent
  │
  ▼
External Service
  │
  └── No response...
          │
          ▼
       Agent stuck
```

A **Timeout Pattern** places an upper bound on how long an operation is allowed to run.

---

## Core Idea

```text id="w7k4mp"
Start Operation
      │
      ▼
   Execute
      │
 ┌────┴─────────────┐
 │                  │
Response          Timeout
 │                  │
 ▼                  ▼
Continue       Retry / Fallback
                  / Escalate
```

Timeouts prevent one slow dependency from consuming the entire execution budget.

---

## Example: Research Agent

A research agent queries an external vulnerability database.

The API has a timeout of 5 seconds:

```text id="m9p3vx"
Agent
  │
  ▼
Vulnerability API
  │
  ├── Response < 5s → Continue
  │
  └── No response → Timeout
                       │
                       ▼
                    Retry
                       │
                  Still failing?
                       │
                       ▼
                    Fallback
```

The agent can then use another source or report that the vulnerability database was unavailable.

---

## Timeout Types

Timeouts should exist at multiple levels.

### Connection Timeout

Maximum time to establish a connection.

### Request Timeout

Maximum time waiting for a service response.

### Tool Timeout

Maximum time allowed for a specific tool.

### Agent Timeout

Maximum time allowed for one agent execution.

### Workflow Timeout

Maximum time allowed for the complete task.

```text id="d8q5nr"
Workflow Timeout
      │
      ├── Agent Timeout
      │      ├── Tool Timeout
      │      └── Tool Timeout
      │
      └── Agent Timeout
```

The overall deadline should always be respected.

---

## Timeout vs Retry

They solve different problems.

```text id="j3v8qm"
Timeout
→ Stop waiting

Retry
→ Try the operation again
```

They are often combined:

```text id="c9m2kp"
Request
  ↓
Timeout after 5s
  ↓
Retry
  ↓
Timeout after 5s
  ↓
Fallback
```

But retries must respect the **overall workflow deadline**.

---

## When to Use

Use timeouts when:

* Calling external services
* Running tools
* Invoking other agents
* Waiting for human/external responses
* Executing long-running operations

---

## When NOT to Use

Almost every distributed operation should have some bounded execution time.

The exception is a deliberately asynchronous operation where the system does not wait for completion.

Instead:

```text id="x7v4nz"
Submit Job
    ↓
Return Job ID
    ↓
Process asynchronously
    ↓
Check status later
```

---

## Production Considerations

Define:

* Connection timeout
* Read/request timeout
* Tool timeout
* Agent timeout
* Workflow deadline
* Cancellation behavior
* Retry policy
* Fallback behavior

Use **deadlines**, not just independent timeouts.

For example:

```text id="p5k8wr"
Workflow deadline = 60s

Remaining time:
  Agent A → 40s
  Agent B → 20s
```

A downstream operation should not start with a timeout larger than the remaining workflow budget.

---

## Trade-offs

| Benefit                    | Cost                              |
| -------------------------- | --------------------------------- |
| Prevents stuck executions  | May terminate slow but valid work |
| Controls latency           | Requires timeout tuning           |
| Protects resources         | Can trigger retries               |
| Improves failure isolation | Adds failure-handling complexity  |

Timeouts that are too short cause unnecessary failures.

Timeouts that are too long allow slow dependencies to degrade the entire system.

---

## Mental Model

> **Timeout = "How long am I willing to wait for this operation?"**

The key principle is:

```text id="v3q9mx"
Every external dependency
        ↓
Must have a bounded waiting time
        ↓
And a defined action after timeout
```
