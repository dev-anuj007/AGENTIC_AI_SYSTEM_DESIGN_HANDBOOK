# Hybrid RAG Pattern

## Problem

Different retrieval methods are good at finding different types of information.

Semantic/vector search is strong at understanding **meaning**, while keyword search is strong at finding **exact terms** such as product names, IDs, error codes, and technical identifiers.

A single retrieval strategy may therefore miss relevant information.

---

## Core Idea

Hybrid RAG combines multiple retrieval strategies and merges their results.

```text id="7q2m1v"
                    Query
                      │
              ┌───────┴───────┐
              ▼               ▼
        Vector Search    Keyword Search
              │               │
              └───────┬───────┘
                      ▼
                Merge / Rank
                      │
                      ▼
                  Reranker
                      │
                      ▼
                    LLM
```

---

## Example: Enterprise Technical Assistant

A developer asks:

> "What caused error `AUTH-4037` after upgrading service `payment-api`?"

The query contains both:

* Semantic intent: *what caused the authentication failure?*
* Exact identifiers: `AUTH-4037`, `payment-api`

The system can retrieve:

```text id="4n8x2c"
Vector Search
→ authentication upgrade documentation
→ related incident reports

Keyword Search
→ exact AUTH-4037 occurrences
→ payment-api deployment logs

        ↓

Combined Results
        ↓
Reranking
        ↓
LLM
```

The combination improves the chance of finding both conceptually relevant and exact-match evidence.

---

## Retrieval Strategies

### Vector Search

Good for:

* Semantic similarity
* Natural-language questions
* Conceptually related content

### Keyword Search

Good for:

* Error codes
* Product names
* IDs
* Version numbers
* Exact technical terms

### Reranking

A reranker can evaluate the combined candidates and select the most relevant context for the LLM.

---

## Result Fusion

The system needs a strategy for combining results from different retrievers.

Common approaches include:

* Score normalization
* Weighted scoring
* Rank-based fusion
* Reranking

The important point is that raw scores from different retrieval systems are not always directly comparable.

---

## When to Use

Use Hybrid RAG when:

* Queries contain exact identifiers and natural language
* Technical documentation is involved
* Keyword and semantic retrieval have complementary strengths
* Search quality from either method alone is insufficient

---

## When NOT to Use

Avoid it when:

* One retrieval strategy already provides sufficient quality
* The corpus is very small
* Additional retrieval complexity provides little benefit
* Latency requirements are extremely strict

---

## Production Considerations

Monitor each retrieval path separately:

```text id="n3r6kp"
Query
 ├── Vector → candidates
 └── Keyword → candidates
          ↓
       Fusion
          ↓
       Reranker
          ↓
        Context
```

Measure:

* Recall
* Precision
* Retrieval latency
* Reranker latency
* Duplicate results
* Source quality
* Final answer quality

Also ensure metadata and authorization filters are applied consistently across **all retrieval paths**.

---

## Trade-offs

| Benefit                          | Cost                    |
| -------------------------------- | ----------------------- |
| Better retrieval coverage        | More infrastructure     |
| Handles exact + semantic queries | Higher latency          |
| More robust search               | More tuning             |
| Works well for technical data    | More complex evaluation |

---

## Mental Model

> **Hybrid RAG = use multiple retrieval strategies because no single retriever is optimal for every query.**

The key question is:

**Does my query contain both semantic meaning and information that requires exact matching?**

If yes, hybrid retrieval is often worth considering.
