# Basic RAG Pattern

## Problem

LLMs have limited knowledge of private, recent, or domain-specific data.

Sending the entire knowledge base to the model is usually impractical because of context limits, latency, and cost.

**Retrieval-Augmented Generation (RAG)** solves this by retrieving relevant information before generating the answer.

---

## Core Idea

```text id="8q4m1c"
User Query
     │
     ▼
  Retrieve
     │
     ▼
Relevant Documents
     │
     ▼
    LLM
     │
     ▼
   Answer
```

The model generates an answer using retrieved external context rather than relying only on its pretrained knowledge.

---

## Example: Enterprise Knowledge Assistant

A company has thousands of internal documents.

A user asks:

> "What is our production database backup policy?"

The system:

```text id="v6k2pd"
Query
  │
  ▼
Embedding
  │
  ▼
Vector Search
  │
  ▼
Relevant Policy Chunks
  │
  ▼
LLM + Retrieved Context
  │
  ▼
Answer
```

Instead of giving the model every company document, the system retrieves only the chunks relevant to the question.

---

## Retrieval Pipeline

A typical RAG system has two phases.

### Indexing

```text id="m8r3ka"
Documents
   ↓
Parse
   ↓
Chunk
   ↓
Embed
   ↓
Vector Database
```

### Querying

```text id="j5p9tx"
Query
  ↓
Embed
  ↓
Similarity Search
  ↓
Top-K Chunks
  ↓
Prompt + Context
  ↓
LLM
```

The quality of both phases affects the final answer.

---

## Key Design Decisions

### Chunking

Chunks should contain enough context to be meaningful without becoming unnecessarily large.

### Embeddings

Embeddings represent documents and queries in a vector space so semantically related content can be retrieved.

### Top-K

Retrieving too few chunks can miss relevant information.

Retrieving too many can introduce noise and increase context cost.

### Metadata Filtering

Production systems often filter retrieval using metadata such as:

```text
tenant_id
document_id
user_id
access_level
document_type
```

This is important for authorization and tenant isolation.

---

## When to Use

Basic RAG works well when:

* Information exists in external documents
* Questions can usually be answered from a small set of relevant documents
* Retrieval is relatively straightforward
* The system does not need complex multi-step investigation

---

## When NOT to Use

Basic RAG becomes insufficient when:

* The answer requires multiple retrieval steps
* The system must decide what to search next
* Different retrieval strategies are required
* Initial retrieval may be wrong and needs correction
* The task requires reasoning across many sources

These cases lead to patterns such as **Agentic RAG, Corrective RAG, Multi-Hop RAG, and Hybrid RAG**.

---

## Production Considerations

A production RAG system should consider:

* Document parsing quality
* Chunking strategy
* Embedding model selection
* Metadata filtering
* Retrieval latency
* Access control
* Reranking
* Context limits
* Citation/provenance
* Retrieval evaluation

Most importantly:

> **Good generation cannot compensate for consistently bad retrieval.**

---

## Trade-offs

| Benefit                                | Cost                                    |
| -------------------------------------- | --------------------------------------- |
| Grounds answers in external data       | Retrieval can return irrelevant context |
| Works with private knowledge           | Requires an ingestion pipeline          |
| Knowledge can be updated independently | Additional latency                      |
| Reduces reliance on model memory       | Retrieval quality becomes critical      |

---

## Mental Model

> **Basic RAG = Retrieve relevant context first, then generate using that context.**

The fundamental pipeline is:

```text
Query → Retrieve → Context → Generate
```
