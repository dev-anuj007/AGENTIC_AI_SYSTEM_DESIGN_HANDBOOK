# Semantic Memory

## Problem

Agents often need to remember **facts, concepts, relationships, and knowledge** independently of when or where that information was learned.

This is **semantic memory**.

---

## Core Idea

```text id="q2k7mv"
Facts / Concepts / Relationships
              │
              ▼
       Semantic Memory
              │
              ▼
        Future Tasks
```

Unlike episodic memory, semantic memory focuses on **what is known**, not the experience through which it was learned.

---

## Example: Enterprise Engineering Assistant

Suppose an engineering organization establishes:

```text id="n8p4xc"
Service:
payment-api

Owner:
Payments Team

Database:
PostgreSQL

Deployment:
AWS ECS

Dependency:
Stripe SDK
```

This information can be stored as semantic memory.

Later, when an engineer asks:

> "Who owns payment-api?"

The agent can retrieve the stored fact directly.

It does not need to remember the conversation where the information was originally discovered.

---

## Semantic vs Episodic Memory

| Semantic Memory        | Episodic Memory            |
| ---------------------- | -------------------------- |
| Facts and knowledge    | Experiences and events     |
| "What is true?"        | "What happened?"           |
| Service ownership      | Previous incident          |
| Architecture knowledge | Previous debugging attempt |
| Relationships          | Actions and outcomes       |

For example:

```text id="v6t3kx"
Semantic:
"payment-api is owned by the Payments Team."

Episodic:
"Last Tuesday, the Payments Team fixed
a payment-api database connection issue."
```

---

## Representation

Semantic memory can be represented using:

* Relational records
* Documents
* Key-value data
* Knowledge graphs
* Vector embeddings

The representation should match the type of knowledge.

For strongly connected entities, a graph can be useful:

```text id="m3r7qn"
payment-api
     │
     ├── owned_by → Payments Team
     ├── runs_on  → AWS ECS
     └── uses     → PostgreSQL
```

---

## Memory Lifecycle

Semantic knowledge should be maintained over time.

```text id="x8p2kv"
Discover
   ↓
Validate
   ↓
Store
   ↓
Retrieve
   ↓
Update
   ↓
Expire / Remove
```

This matters because facts can change.

For example:

```text id="f1n6vy"
Old Owner → Team A

Organization changes

New Owner → Team B
```

The system must update the stored fact rather than returning conflicting information indefinitely.

---

## When to Use

Use semantic memory when:

* Stable facts are reused across tasks
* Agents need domain knowledge
* Relationships between entities matter
* Information should outlive individual conversations
* The same knowledge is repeatedly queried

---

## When NOT to Use

Do not use semantic memory for information whose primary value is the **sequence of events or actions**.

For example:

> "What happened during yesterday's incident?"

That belongs more naturally in episodic memory.

---

## Production Considerations

Important concerns include:

* Source/provenance
* Versioning
* Freshness
* Conflict resolution
* Access control
* Tenant isolation
* Update and deletion policies
* Retrieval relevance

Whenever possible, store where a fact came from so the system can validate or refresh it.

---

## Mental Model

> **Semantic memory = knowledge about what is true or known.**

The key question is:

**"Is this information a reusable fact or relationship, independent of the specific experience that produced it?"**

If yes, semantic memory is the appropriate abstraction.
