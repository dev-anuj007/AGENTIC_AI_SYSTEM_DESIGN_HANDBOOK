# Multi-Hop RAG Pattern

## Problem

Some questions cannot be answered from a single document or retrieval step.

The system must retrieve information, use it to identify the next piece of information required, and continue until enough evidence is collected.

This is **Multi-Hop RAG**.

---

## Core Idea

```text id="m8k3qv"
Query
  │
  ▼
Retrieve → Evidence 1
              │
              ▼
         Next Question
              │
              ▼
         Retrieve → Evidence 2
              │
              ▼
         Combine Evidence
              │
              ▼
             Answer
```

Each retrieval **hop depends on information discovered in the previous hop**.

---

## Example: Enterprise Security Investigation

Suppose the user asks:

> "Which customer-facing services are running a vulnerable version of library X?"

The answer requires connecting multiple sources:

```text id="f5n2rc"
Hop 1
Search vulnerability database
        ↓
Library X affected versions
        ↓
Hop 2
Query asset inventory
        ↓
Services using library X
        ↓
Hop 3
Check deployed versions
        ↓
Affected services
        ↓
Generate assessment
```

The output of one retrieval step determines what the system searches for next.

---

## Why Not Basic RAG?

Basic RAG might retrieve:

```text
Library X vulnerability
```

But that does not directly answer:

```text
Which of our services are affected?
```

The system must traverse a chain:

```text
Vulnerability
    ↓
Library
    ↓
Service
    ↓
Version
    ↓
Affected Service
```

This is the defining characteristic of multi-hop retrieval.

---

## Multi-Hop vs Agentic RAG

These concepts overlap but describe different things.

| Multi-Hop RAG                                 | Agentic RAG                                     |
| --------------------------------------------- | ----------------------------------------------- |
| Focuses on multiple dependent retrieval steps | Focuses on agent-controlled retrieval decisions |
| Retrieval chain is the key idea               | Autonomy is the key idea                        |
| Can be deterministic                          | Usually dynamic                                 |
| May use an agent                              | Does not require an agent                       |

A multi-hop system can therefore be implemented as a predefined retrieval workflow or as an agent that dynamically determines each hop.

---

## When to Use

Use Multi-Hop RAG when:

* The answer requires information from multiple sources
* One retrieval result identifies the next search target
* Relationships between entities matter
* Questions require investigation across several steps

---

## When NOT to Use

Avoid it when:

* One retrieval step normally provides the answer
* The hops are independent and can run in parallel
* A deterministic database query is simpler
* Additional retrieval does not improve answer quality

If the searches are independent, **Fan-Out/Fan-In** may be more appropriate.

---

## Production Considerations

Control:

* Maximum hops
* Query generation
* Retrieval quality per hop
* Context accumulation
* Timeouts
* Cost/token budgets
* Evidence provenance
* Failure handling

Track each hop independently:

```text
query → retrieval → evidence → next_query
```

This makes incorrect reasoning and retrieval failures easier to diagnose.

---

## Trade-offs

**Benefits**

* Handles complex questions
* Connects information across sources
* Supports investigative workflows

**Costs**

* Higher latency
* More retrieval calls
* More accumulated context
* Greater failure surface
* Harder evaluation

---

## Mental Model

> **Multi-Hop RAG = the answer is discovered through a chain of dependent retrieval steps.**

The key question is:

**Does the result of this search determine what I need to search for next?**

If yes, you likely have a multi-hop retrieval problem.
