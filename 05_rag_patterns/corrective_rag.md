# Corrective RAG Pattern

## Problem

A RAG system may retrieve documents that are:

* Irrelevant
* Incomplete
* Low quality
* Insufficient to answer the query

A basic RAG pipeline may pass those results directly to the LLM.

**Corrective RAG (CRAG)** introduces a validation step before generation.

---

## Core Idea

```text id="j7m2qx"
Query
  │
  ▼
Retrieve
  │
  ▼
Evaluate Retrieval
  │
  ├── Good ───────→ Generate
  │
  └── Poor
        │
        ▼
   Correct Retrieval
        │
        ├── Rewrite Query
        ├── Retrieve Again
        └── Use Alternative Source
                │
                ▼
              Generate
```

The key idea is:

> **Do not blindly trust the first retrieval result.**

---

## Example: Enterprise Security Assistant

A user asks:

> "Does our payment service use a vulnerable version of library X?"

The first retrieval searches internal documentation but returns only an outdated architecture document.

A corrective RAG system evaluates the retrieved evidence and determines that it is insufficient.

It can then:

```text id="p8r4vz"
Initial Retrieval
      ↓
Old architecture document
      ↓
Evidence insufficient
      ↓
Rewrite / refine query
      ↓
Search asset inventory
      ↓
Find current dependency version
      ↓
Generate answer
```

The system therefore uses **retrieval quality to determine the next action**.

---

## Retrieval Evaluation

The evaluator can use:

* Similarity score thresholds
* Metadata/source quality
* Rule-based checks
* LLM-based relevance grading
* Required evidence checks

For example:

```text id="k2v9sa"
Retrieved Context
      ↓
Is it relevant?
      ↓
Is it recent enough?
      ↓
Does it contain required evidence?
```

A similarity score alone should not necessarily determine whether retrieval is sufficient.

---

## Correction Strategies

When retrieval is poor, the system can:

### Query Rewrite

Transform the original query into a more precise search query.

### Additional Retrieval

Search again using different parameters or filters.

### Alternative Source

Use another knowledge source such as a database, API, or web search.

### Hybrid Retrieval

Combine keyword and semantic retrieval when one retrieval method is insufficient.

---

## When to Use

Use Corrective RAG when:

* Retrieval quality varies
* Incorrect retrieval can produce harmful answers
* Documents may be stale
* Multiple knowledge sources exist
* Evidence quality matters

---

## When NOT to Use

Avoid it when:

* Retrieval is highly reliable
* A simple RAG pipeline already meets requirements
* The additional evaluation latency is unacceptable
* The application does not require evidence validation

---

## Production Considerations

Define explicit correction policies:

```text id="s4c8ny"
Good retrieval
    → Generate

Poor retrieval
    → Correct

Still insufficient
    → Escalate / abstain
```

Do not allow unlimited correction loops.

Use:

* Maximum retrieval attempts
* Query rewrite limits
* Timeouts
* Cost budgets
* Source prioritization
* Evidence thresholds
* Abstention/fallback behavior

For sensitive applications, **"I don't have enough evidence"** can be safer than generating an unsupported answer.

---

## Trade-offs

| Benefit                      | Cost                      |
| ---------------------------- | ------------------------- |
| Better retrieval reliability | Additional latency        |
| Reduces unsupported answers  | More model/tool calls     |
| Handles poor retrieval       | More complex architecture |
| Supports multiple sources    | Requires evaluation logic |

---

## Basic RAG vs Corrective RAG

```text
Basic RAG
Query → Retrieve → Generate

Corrective RAG
Query → Retrieve → Evaluate
                  │
             ┌────┴────┐
            Good      Poor
             │          │
          Generate    Correct
```

---

## Mental Model

> **Corrective RAG = retrieve, verify, and correct before generating.**

The important design question is:

**What should the system do when the retrieved evidence is not good enough?**
