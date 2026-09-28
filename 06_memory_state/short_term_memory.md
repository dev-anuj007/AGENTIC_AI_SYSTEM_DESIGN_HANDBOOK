# Short-Term Memory

## Problem

An agent needs to remember what has happened during the current interaction.

Without short-term memory, every step would have to reconstruct context from scratch.

Short-term memory maintains the **current conversation or execution state**.

---

## Core Idea

```text id="k4m8zq"
User Request
     │
     ▼
 Agent Step 1
     │
     ▼
 Agent Step 2
     │
     ▼
 Agent Step 3
     │
     ▼
 Final Response

      ▲
      │
  Short-Term State
```

The state persists across steps within the active execution.

---

## Example: Customer Support Agent

A customer says:

> "My order hasn't arrived. The tracking number is 12345."

The agent retrieves the order and discovers:

```text id="5y7n2c"
State
├── user_message
├── order_id = 12345
├── order_status = delayed
├── customer_intent = delivery_issue
└── tool_results
```

When the agent later calls the shipping system, it can use the existing state rather than asking the customer for the tracking number again.

---

## What Short-Term Memory Contains

Typical state includes:

* Conversation messages
* Current task
* Tool results
* Intermediate reasoning outputs
* Current agent/workflow status
* User-provided context
* Pending actions

Not everything needs to remain forever. State should be limited to what is relevant to the active task.

---

## Conversation History vs State

They are related but different.

```text
Conversation History
→ What was said

State
→ What the system currently knows and needs to execute
```

For example:

```text
Conversation:
"My order hasn't arrived."

State:
{
  "intent": "delivery_issue",
  "order_id": "12345",
  "status": "delayed"
}
```

The state is an execution-oriented representation of the conversation and tool activity.

---

## Persistence

Short-term memory can be:

### In-Memory

Useful for a single execution.

```text
Request → Agent → State → Response
```

### Persistent

Useful when execution can pause and resume.

```text
Request
  ↓
State Store
  ↓
Agent
  ↓
Updated State
  ↓
State Store
```

Persistent state is especially important for long-running workflows.

---

## When to Use

Use short-term memory when:

* Multiple steps depend on previous results
* Tools are called during execution
* Conversations span multiple turns
* Workflows may pause and resume
* Agents need current execution context

---

## When NOT to Use

Do not keep everything indefinitely.

Avoid putting:

* Unrelated historical conversations
* Large documents
* Redundant tool responses
* Sensitive information without a requirement

into active state.

Excessive state increases storage, context, latency, and cost.

---

## Production Considerations

A production system should define:

* State schema
* State ownership
* Persistence strategy
* Expiration/TTL
* Size limits
* Serialization format
* Access control
* Checkpointing
* Recovery behavior

For long conversations, use summarization or selective context retrieval instead of continuously sending the complete history to the model.

---

## Trade-offs

| Benefit                            | Cost                            |
| ---------------------------------- | ------------------------------- |
| Maintains execution context        | Increases context size          |
| Enables multi-step tasks           | Requires state management       |
| Supports resume/recovery           | Persistence adds infrastructure |
| Improves conversational continuity | Stale state can cause errors    |

---

## Mental Model

> **Short-term memory is the agent's working state for the current interaction or execution.**

Think of it as:

```text
Short-Term Memory
=
"What does the agent need to remember right now to continue the task?"
```
