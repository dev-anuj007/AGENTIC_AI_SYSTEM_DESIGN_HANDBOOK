# Durable State

## Problem

Agent executions can be long-running, expensive, or interrupted.

If the process crashes or the user pauses the task, the agent should not necessarily start from the beginning.

**Durable state** allows execution state to survive failures, restarts, and interruptions.

---

## Core Idea

```text id="r7m3kx"
Start
  │
  ▼
Execute
  │
  ▼
Checkpoint
  │
  ▼
Execute
  │
  X ← Failure / Shutdown
  │
  ▼
Resume from Checkpoint
  │
  ▼
Continue
```

The system periodically persists the state required to continue execution.

---

## Example: Long-Running Research Agent

Suppose an agent is researching a company across multiple sources:

```text id="n5q8vz"
Research
   │
   ├── Company information ✓
   ├── Financial data ✓
   ├── Security research ✓
   ├── Market research ← current
   └── Final analysis
```

After completing security research, the system saves a checkpoint.

If the process crashes during market research:

```text id="x3k9pd"
Restart
   ↓
Load Checkpoint
   ↓
Skip completed work
   ↓
Continue Market Research
```

The system does not need to repeat the completed steps.

---

## Durable State vs Short-Term Memory

These concepts are related but different.

| Short-Term Memory         | Durable State                      |
| ------------------------- | ---------------------------------- |
| Current working context   | Persisted execution state          |
| May exist only in memory  | Designed to survive interruption   |
| Supports reasoning        | Supports recovery/resume           |
| Conversation/task context | Checkpoints and execution progress |

Short-term memory answers:

> "What does the agent currently know?"

Durable state answers:

> "What execution state must survive a restart?"

---

## What Should Be Persisted?

Typically:

```text id="h2v7qm"
task_id
goal
current_step
completed_steps
tool_results
agent_outputs
pending_actions
retry_count
execution_status
timestamps
```

Do not blindly persist every intermediate token or temporary object.

Persist what is required to safely resume.

---

## Checkpoint Strategy

Checkpoints can be created:

* After important workflow steps
* Before expensive operations
* Before external side effects
* At regular intervals
* After successful tool execution

The correct frequency depends on the cost of recomputing work versus the cost of storing state.

---

## Idempotency

Durable state must work together with **idempotency**.

Consider:

```text
Step completed
     ↓
External API called
     ↓
Process crashes before checkpoint
     ↓
System retries step
```

The external operation may execute twice.

For side-effecting operations, use idempotency keys or transactional mechanisms where possible.

---

## When to Use

Use durable state when:

* Tasks run for a long time
* Work can be interrupted
* External systems may be unreliable
* Expensive computation should not be repeated
* Human approval can pause execution
* Work needs to resume later

---

## When NOT to Use

It may be unnecessary when:

* Execution is short
* Recomputing the task is cheap
* No interruption or resume capability is required
* The workflow has no meaningful intermediate state

---

## Production Considerations

A production implementation should handle:

* Atomic checkpoints
* State versioning
* State corruption
* Concurrent executions
* Recovery after crashes
* Expiration/retention
* Encryption and access control
* Idempotent side effects

A checkpoint should represent a **consistent state**, not a partially written execution.

---

## Trade-offs

| Benefit               | Cost                        |
| --------------------- | --------------------------- |
| Crash recovery        | Storage overhead            |
| Resume long tasks     | State management complexity |
| Avoids recomputation  | Checkpoint coordination     |
| Supports human pauses | Versioning requirements     |

---

## Mental Model

> **Durable state turns an agent execution from a temporary process into a resumable process.**

The key question is:

**"If this process dies right now, what information would I need to safely continue?"**

That information belongs in durable state.
