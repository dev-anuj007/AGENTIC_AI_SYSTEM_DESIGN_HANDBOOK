# Agent Tracing

## Problem

Agentic systems are difficult to debug because execution is dynamic.

A single user request may involve:

```text
User Request
   ↓
Agent
   ↓
LLM Decision
   ↓
Tool
   ↓
Observation
   ↓
LLM Decision
   ↓
RAG
   ↓
Another Tool
   ↓
Final Response
```

Traditional request logs often cannot explain **why** the agent took a particular path.

Agent tracing captures the complete execution path.

---

## Core Idea

A trace represents one logical agent execution.

```text id="j7k3pq"
Trace: request_123
│
├── Agent Run
│   ├── LLM Call
│   ├── Tool Call
│   │   └── API Response
│   ├── LLM Call
│   ├── Retrieval
│   └── Tool Call
│
└── Final Response
```

Each operation becomes a **span** inside the trace.

---

## Example: Security Investigation Agent

A user asks:

> "Investigate why our API is returning 500 errors."

The agent may execute:

```text id="x4n8vz"
Trace: incident_123
       │
       ▼
Agent Run
       │
       ├── LLM Decision
       │
       ├── Logs Tool
       │      └── 500 errors found
       │
       ├── LLM Decision
       │
       ├── Database Tool
       │      └── deployment changed
       │
       ├── LLM Decision
       │
       └── Final Diagnosis
```

Without tracing, you may only see:

```text
Request completed
```

With tracing, you can determine:

* Which tools were called
* How long each operation took
* Which LLM calls occurred
* What failed
* How many iterations occurred
* Where latency increased
* Which path the agent followed

---

## Trace Structure

A useful trace hierarchy is:

```text id="c5r9mk"
Trace
 │
 └── Agent Run
      │
      ├── LLM Span
      │
      ├── Tool Span
      │
      ├── Retrieval Span
      │
      ├── Agent / Subgraph Span
      │
      └── LLM Span
```

Each span should contain relevant metadata.

```text id="m8v2qa"
span_id
parent_span_id
trace_id
operation
start_time
duration
status
error
agent
model
tool
```

---

## Correlation

Every operation belonging to one execution should share the same trace ID.

```text id="a3f7kp"
User Request
trace_id = abc123
       │
       ├── Agent
       │   trace_id = abc123
       │
       ├── Tool
       │   trace_id = abc123
       │
       └── Database
           trace_id = abc123
```

This allows engineers to follow one request across distributed services.

---

## What to Capture

Important agent-level signals include:

### Execution

* Trace ID
* Agent ID
* Workflow ID
* User/request ID
* Iteration count
* Execution status

### LLM

* Model
* Prompt/version
* Token usage
* Latency
* Response status

### Tools

* Tool name
* Arguments
* Duration
* Result status
* Error type
* Retry count

### Retrieval

* Query
* Retriever
* Top-k
* Documents retrieved
* Reranker
* Retrieval latency

Avoid logging sensitive data blindly.

---

## Practical Architecture

```text id="q6w4mx"
Agent Runtime
     │
     ├── LLM
     ├── Tools
     ├── RAG
     └── Subgraphs
          │
          ▼
     Trace Context
          │
          ▼
   Observability Layer
          │
     ┌────┼────┐
     ▼    ▼    ▼
   Traces Metrics Logs
```

A common production approach is to use **OpenTelemetry** for distributed tracing and connect the traces to an observability backend.

---

## Failure Investigation

Suppose the agent suddenly becomes slow.

Tracing may reveal:

```text id="p2m7rx"
Total: 12.4s

Agent              0.2s
LLM Call #1        1.1s
Search Tool        0.4s
LLM Call #2        1.2s
Database Tool      8.9s  ← bottleneck
Final LLM          0.6s
```

The trace immediately identifies the database call as the latency bottleneck.

Without span-level tracing, the application may only report:

```text
Request latency = 12.4s
```

---

## Tracing vs Logging

They complement each other.

| Tracing                         | Logging                         |
| ------------------------------- | ------------------------------- |
| Shows execution path            | Shows individual events         |
| Connects distributed operations | Provides detailed event data    |
| Measures span latency           | Captures contextual information |
| Uses trace/span IDs             | Uses structured log fields      |

A production agentic system generally needs both.

---

## Production Considerations

A tracing system should handle:

* Distributed trace propagation
* Trace/span IDs
* Sampling
* PII and secret redaction
* Token/cost metadata
* Tool arguments safely
* Error tracking
* Trace retention
* High-cardinality fields
* Tenant isolation
* Agent/subgraph hierarchy

Never log secrets, credentials, API keys, or unrestricted sensitive prompts/responses.

---

## When to Use

Agent tracing is essential when:

* Agents call multiple tools
* Workflows are distributed
* Multiple agents collaborate
* RAG is involved
* Debugging dynamic behavior
* Latency/cost optimization matters
* Production failures need investigation

---

## Trade-offs

| Benefit                        | Cost                       |
| ------------------------------ | -------------------------- |
| Debugs dynamic execution       | Storage cost               |
| Identifies bottlenecks         | Instrumentation complexity |
| Supports distributed debugging | Privacy considerations     |
| Enables agent analytics        | Trace volume               |

---

## Mental Model

> **Logs tell you what happened. Traces tell you how the entire agent execution happened.**

```text id="e8r3nv"
User Request
     ↓
Trace
     ↓
Agent
 ┌───┼────┬────┐
 ↓   ↓    ↓    ↓
LLM Tool RAG Subgraph
 └───┴────┴────┘
          ↓
      Final Result
```
