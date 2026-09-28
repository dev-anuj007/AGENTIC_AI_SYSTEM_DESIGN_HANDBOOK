# Episodic Memory

## Problem

Long-term memory can store facts and preferences, but agents may also need to remember **what happened during previous tasks**.

This is episodic memory.

It captures experiences, events, and outcomes rather than just static facts.

---

## Core Idea

```text id="q7m3cx"
Past Task
   │
   ├── What happened
   ├── What actions were taken
   ├── What result occurred
   └── What was learned
          │
          ▼
   Episodic Memory
          │
          ▼
Future Task
```

The agent can use previous experiences to improve future execution.

---

## Example: Customer Support Agent

Suppose an agent previously handled a customer issue:

```text id="z8p2nk"
Incident:
Payment failed after card replacement

Actions:
1. Checked payment status
2. Verified card token
3. Requested payment retry

Outcome:
Payment succeeded
```

Later, a similar issue occurs.

The agent can retrieve the previous episode:

```text id="r3v6ht"
New Issue
   ↓
Find Similar Past Episode
   ↓
Reuse Relevant Experience
   ↓
Take Action
```

The agent is not simply remembering that the customer uses a particular payment method.

It is remembering **how a previous situation unfolded**.

---

## Episodic vs Semantic Memory

The distinction is important:

| Episodic Memory      | Semantic Memory            |
| -------------------- | -------------------------- |
| Remembers events     | Remembers facts            |
| "What happened?"     | "What is true?"            |
| Task history         | Knowledge                  |
| Actions and outcomes | Concepts and relationships |
| Experience-oriented  | Fact-oriented              |

Example:

```text
Semantic:
"OAuth tokens expire after 1 hour."

Episodic:
"During the previous OAuth incident,
refresh-token rotation fixed the authentication failure."
```

---

## What an Episode Should Contain

A useful episode might contain:

```text id="c5q9rm"
episode_id
goal
context
actions
tool_calls
observations
outcome
failure_reason
final_result
timestamp
```

Avoid storing unnecessary execution details. Store the information that could help with future decisions.

---

## Retrieval

Episodes should usually be retrieved based on relevance.

```text id="w4n7xp"
Current Task
     ↓
Similarity / Metadata Search
     ↓
Relevant Episodes
     ↓
Agent Context
```

Useful retrieval signals can include:

* Semantic similarity
* Task type
* Outcome
* Domain
* Recency
* User/project scope

---

## When to Use

Use episodic memory when:

* Previous task outcomes can inform future decisions
* Agents repeatedly perform similar tasks
* Past failures are useful for avoiding repetition
* Work is long-running
* Historical context matters

---

## When NOT to Use

Avoid episodic memory when:

* Past executions have little relevance to future tasks
* A static fact is sufficient
* Historical details create more noise than value
* Storing the full execution trace provides no future benefit

---

## Production Considerations

Important concerns include:

* Episode relevance
* Deduplication
* Retention policies
* Access control
* Stale or contradictory experiences
* Privacy
* Outcome quality

An unsuccessful episode should not automatically become a trusted strategy.

For example:

```text id="j9p4vz"
Past Action
    ↓
Failed
    ↓
Stored as failed experience
    ↓
Future agent learns to avoid it
```

The outcome should therefore be part of the memory.

---

## Mental Model

> **Episodic memory = remembering experiences.**

A useful question is:

**"Have I encountered a similar situation before, and what happened when I handled it?"**
