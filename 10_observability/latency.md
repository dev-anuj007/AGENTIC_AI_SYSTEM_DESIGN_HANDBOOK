# Agent Latency

## Problem

Agentic systems can become slow because one request may involve multiple sequential operations:

```text id="z4q7np"
User Request
    ↓
LLM
    ↓
Tool
    ↓
LLM
    ↓
RAG
    ↓
Tool
    ↓
LLM
    ↓
Response
```

Even if each individual operation is reasonably fast, the total latency can become large.

---

## Core Idea

Measure latency at every execution boundary and identify the critical path.

```text id="c8m2vx"
Total Latency
      │
      ├── LLM #1
      ├── Tool
      ├── LLM #2
      ├── RAG
      └── LLM #3
```

The key question is:

> **Where is the time actually being spent?**

---

## Example: Research Agent

Suppose one research request takes:

```text id="n6p4ka"
LLM #1              800ms
Web Search          400ms
LLM #2              900ms
Database Query    2,500ms
LLM #3              700ms
──────────────────────────
Total              5,300ms
```

The database query is the largest latency contributor.

Without component-level measurement, the only visible metric might be:

```text id="w8q3ry"
Request latency = 5.3 seconds
```

Tracing reveals where the time is actually spent.

---

## Latency Hierarchy

Measure latency at multiple levels:

```text id="j2v7mc"
Request
  │
  └── Agent Run
       │
       ├── LLM Span
       ├── Tool Span
       ├── Retrieval Span
       └── Sub-Agent Span
```

Each span should have:

```text id="t3k9pq"
start_time
end_time
duration
status
```

---

## Sequential vs Parallel Latency

This is especially important for agent workflows.

### Sequential

```text id="8n5m2v"
A → B → C

Total ≈ A + B + C
```

### Parallel

```text id="q7r4mx"
      ┌→ B ─┐
A ────┤     ├→ D
      └→ C ─┘

Total ≈ A + max(B, C) + D
```

Independent operations should be parallelized when doing so does not introduce unacceptable complexity or resource usage.

---

## Critical Path

The **critical path** is the sequence of dependent operations that determines the minimum possible execution time.

```text id="p5k8qa"
A ──→ B ──→ D
      │
      └──→ C
```

If:

```text
A = 100ms
B = 1000ms
C = 2000ms
D = 300ms
```

then:

```text
A → B → D = 1400ms
A → C → D = 2400ms
```

The longer dependency path determines the critical-path latency.

---

## Agent-Specific Latency

Track:

```text id="d8x3vp"
Agent latency
├── Time to first LLM response
├── Total LLM time
├── Tool latency
├── Retrieval latency
├── Waiting / approval time
├── Retry time
└── Total execution time
```

This helps distinguish **compute latency** from **waiting latency**.

---

## Latency and Streaming

For interactive agents, total latency is not the only metric.

Important metrics include:

```text id="m4q9zs"
Time to First Token
        ↓
Time to First Useful Output
        ↓
Time to Final Response
```

Streaming can improve perceived responsiveness even when total execution time remains unchanged.

---

## Latency Budgets

Set an execution deadline.

```text id="v6r2kp"
Request
  │
  ▼
Budget = 10 seconds
  │
  ├── LLM        2s
  ├── Tool       3s
  ├── RAG        1s
  └── Remaining  4s
```

Once the deadline is approaching, the system can:

* Skip optional work
* Use a fallback
* Reduce retrieval depth
* Switch to a faster model
* Return partial results
* Terminate execution

---

## Latency vs Reliability

Retries can increase latency:

```text id="k8m3vx"
Tool
 │
 X failure
 │
 ▼
Retry
 │
 ▼
Success
```

Therefore monitor:

```text
Base Latency
+
Retry Latency
+
Fallback Latency
```

A highly reliable agent can still provide a poor user experience if recovery behavior causes excessive latency.

---

## Production Considerations

Track:

* p50 latency
* p95 latency
* p99 latency
* LLM latency
* Tool latency
* Retrieval latency
* Queue/wait latency
* Retry latency
* Human-approval latency
* Total execution time
* Time to first token

Combine these metrics with traces to identify the source of latency.

---

## When to Use

Latency tracking is essential when:

* Agents interact with users synchronously
* Multiple tools are involved
* Workflows contain many steps
* SLAs exist
* External dependencies are slow
* Parallelism or caching is being optimized

---

## Trade-offs

| Optimization        | Trade-off                          |
| ------------------- | ---------------------------------- |
| Parallel execution  | More resource usage                |
| Faster model        | Potential quality difference       |
| Caching             | Stale results                      |
| Fewer tool calls    | Potentially less information       |
| Smaller context     | Potentially less reasoning context |
| Aggressive timeouts | More incomplete requests           |

---

## Mental Model

> **Don't optimize total latency blindly. Find the critical path, then optimize the operations that actually determine it.**

```text id="r7m2qx"
Request
   ↓
Trace
   ↓
Find Critical Path
   ↓
Identify Bottleneck
   ↓
Parallelize / Cache / Optimize / Fallback
   ↓
Measure Again
```
