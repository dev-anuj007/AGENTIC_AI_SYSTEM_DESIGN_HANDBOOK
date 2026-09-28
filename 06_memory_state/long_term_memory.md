# Long-Term Memory

## Problem

Short-term memory only preserves information needed for the current interaction.

An agent may also need to remember information across **different conversations or executions**.

This is the purpose of long-term memory.

---

## Core Idea

```text id="5j8n3p"
Conversation 1 ──┐
Conversation 2 ──┼──→ Long-Term Memory
Conversation 3 ──┘          │
                            ▼
                     Future Execution
```

Information is stored beyond the lifetime of the current execution and can be retrieved later.

---

## Example: Coding Assistant

A developer repeatedly works on the same project.

During one session, they establish:

```text id="8q3v6m"
Project preferences:
- Python
- FastAPI
- PostgreSQL
- pytest
- Repository uses snake_case
```

A future conversation can retrieve these preferences instead of requiring the developer to explain them again.

```text id="v2m7kx"
New Conversation
      │
      ▼
Retrieve Relevant Memory
      │
      ▼
Agent Context
      │
      ▼
Response
```

The important part is **relevance**: the system should retrieve memories that help with the current task rather than injecting everything it has stored.

---

## What Can Be Stored?

Long-term memory can contain:

* User preferences
* Project conventions
* Previously established facts
* Important decisions
* Learned information
* Persistent task context

Memory should be treated as data with a lifecycle, not as an unlimited transcript.

---

## Storage and Retrieval

A typical architecture is:

```text id="0c5n8x"
Agent
  │
  ├── Read Memory ──→ Memory Store
  │
  └── Write Memory ─→ Memory Store
```

The memory store could use:

* Relational databases
* Key-value stores
* Document databases
* Vector databases
* Specialized memory systems

The storage mechanism should match the type of memory being stored.

---

## Memory Lifecycle

A production memory system needs more than storage.

```text id="s7q2mp"
Capture
  ↓
Validate
  ↓
Store
  ↓
Retrieve
  ↓
Use
  ↓
Update / Expire
```

Important questions include:

* Should this information be remembered?
* Is it still valid?
* Who is allowed to access it?
* When should it expire?
* Can the user modify or delete it?

---

## When to Use

Use long-term memory when:

* Information remains useful across sessions
* Personalization matters
* Agents work on long-running projects
* Previous decisions should influence future tasks
* Repeated context would otherwise need to be re-established

---

## When NOT to Use

Do not store information simply because it appeared in a conversation.

Avoid persistent memory when information is:

* Temporary
* Irrelevant to future tasks
* Highly sensitive without a legitimate requirement
* Likely to become stale quickly

---

## Production Considerations

Long-term memory requires:

* Access control
* Tenant/user isolation
* Data lifecycle policies
* TTL or expiration where appropriate
* Memory relevance scoring
* Conflict resolution
* Auditability
* User control

A particularly important problem is **stale memory**.

For example:

```text
Old preference → stored memory
        ↓
User changes preference
        ↓
Old memory becomes incorrect
```

The system must be able to update or invalidate previous information.

---

## Short-Term vs Long-Term

| Short-Term                 | Long-Term                   |
| -------------------------- | --------------------------- |
| Current execution          | Across executions           |
| Working context            | Persistent knowledge        |
| Often temporary            | Persistent                  |
| Tool results/current state | Preferences/facts/decisions |
| Supports current task      | Supports future tasks       |

---

## Mental Model

> **Short-term memory answers: "What do I need to remember right now?"**

> **Long-term memory answers: "What should I remember for future interactions?"**

The key design principle is:

**Persist useful information deliberately; do not turn every conversation into permanent memory.**
