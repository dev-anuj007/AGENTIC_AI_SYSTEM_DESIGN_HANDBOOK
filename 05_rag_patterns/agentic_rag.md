# Agentic RAG Pattern

## Problem

Basic RAG follows a mostly fixed flow:

```text
Query → Retrieve → Generate
```

This works when the relevant information can be retrieved with one search.

Some questions require the system to decide **what to search, whether the results are sufficient, and what to search next**.

This is where **Agentic RAG** fits.

---

## Core Idea

The LLM becomes responsible for controlling the retrieval process.

```text id="c8q2mx"
User Query
     │
     ▼
   Agent
     │
     ├── Search Knowledge Base
     ├── Search Web
     ├── Search Database
     └── Search Again
            │
            ▼
      Evaluate Results
            │
       ┌────┴────┐
       │         │
   Sufficient   Insufficient
       │         │
       ▼         └──→ Retrieve Again
    Generate
```

Retrieval becomes a **dynamic tool** rather than a fixed pipeline step.

---

## Example: Enterprise Security Investigation

Suppose a security analyst asks:

> "Is our authentication service affected by the recent vulnerability, and what should we do?"

A basic RAG system might retrieve documents related to the vulnerability and immediately generate an answer.

An Agentic RAG system can reason through the investigation:

```text id="g7k1rp"
Question
   ↓
Search vulnerability database
   ↓
Identify affected software/version
   ↓
Search internal asset inventory
   ↓
Check deployed version
   ↓
Search internal security policies
   ↓
Determine remediation requirements
   ↓
Generate assessment
```

If the internal asset search shows insufficient information, the agent can perform another retrieval rather than generating an answer from incomplete evidence.

---

## Retrieval as a Tool

The agent may have access to multiple retrieval tools:

```text id="m4p6vz"
Agent
 ├── search_internal_docs()
 ├── search_vulnerabilities()
 ├── query_asset_inventory()
 └── search_web()
```

The agent decides which tool to call based on the current state of the investigation.

---

## Basic RAG vs Agentic RAG

| Basic RAG                   | Agentic RAG                        |
| --------------------------- | ---------------------------------- |
| Fixed retrieval pipeline    | Dynamic retrieval                  |
| Usually one retrieval cycle | Multiple retrieval cycles possible |
| Application controls flow   | Agent controls retrieval decisions |
| Lower complexity            | Higher complexity                  |
| More predictable            | More flexible                      |
| Good for direct questions   | Good for investigations            |

Agentic RAG should not automatically replace Basic RAG. The additional reasoning loop introduces latency, cost, and more failure modes.

---

## When to Use

Use Agentic RAG when:

* Questions require multiple searches
* The next search depends on previous results
* Multiple knowledge sources exist
* Retrieval needs iterative refinement
* The system must investigate rather than simply answer

---

## When NOT to Use

Avoid it when:

* One retrieval step normally answers the question
* Latency must be extremely low
* The retrieval path is predictable
* A deterministic workflow is sufficient

In those cases, Basic RAG is simpler.

---

## Production Considerations

Control the agent with:

* Maximum retrieval iterations
* Tool-call limits
* Token/cost budgets
* Retrieval timeouts
* Source authorization
* Query validation
* Result quality checks
* Full retrieval tracing

Also prevent the agent from repeatedly searching for slightly different versions of the same query.

---

## Trade-offs

**Benefits**

* Handles complex questions
* Can combine multiple sources
* Can refine poor retrieval
* Supports multi-step investigation

**Costs**

* Higher latency
* Higher token/tool cost
* More complex debugging
* Greater risk of retrieval loops
* More difficult evaluation

---

## Mental Model

> **Basic RAG:** the application decides what to retrieve.

> **Agentic RAG:** the agent decides what to retrieve next.

The key shift is:

```text id="z4w8nc"
Fixed Retrieval
      ↓
Dynamic Retrieval Decisions
```
