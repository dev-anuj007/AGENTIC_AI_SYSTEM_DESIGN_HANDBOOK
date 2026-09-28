# Agent Cost Tracking

## Problem

Agentic systems can generate unpredictable costs because one user request may trigger:

```text
User Request
   ↓
LLM Call
   ↓
Tool
   ↓
LLM Call
   ↓
RAG
   ↓
LLM Call
   ↓
Another Tool
   ↓
...
```

Unlike a traditional API, cost can increase with:

* More agent iterations
* More LLM calls
* Larger contexts
* More tool calls
* More retrieved documents
* More expensive models

Therefore, cost should be treated as an observable runtime metric.

---

## Core Idea

Track cost at every execution boundary and aggregate it to the level that matters.

```text id="w5k9rp"
LLM Call
   │
   ├── Input Tokens
   ├── Output Tokens
   └── Model Cost
          │
Tool Call ─┤
   │       │
   └── API Cost
          │
          ▼
       Agent Run
          │
          ▼
      Workflow Cost
          │
          ▼
      Tenant / User
```

The key question is:

> **How much did this agent execution cost, and what caused the cost?**

---

## Example: Research Agent

Suppose a research agent performs:

```text id="v3m8qa"
User Request
    │
    ├── LLM Call #1       $0.02
    ├── Web Search        $0.01
    ├── LLM Call #2       $0.03
    ├── Vector Search     $0.00
    ├── LLM Call #3       $0.04
    └── Final LLM Call    $0.05
                         ───────
                         Total $0.15
```

A single request costing `$0.15` may look acceptable.

But at:

```text
100,000 requests/day
```

the same behavior becomes approximately:

```text
$0.15 × 100,000 = $15,000/day
```

Cost observability makes this behavior visible before it becomes a scaling problem.

---

## Cost Breakdown

Track at least:

```text
LLM
├── Input tokens
├── Output tokens
├── Model
└── Number of calls

Tools
├── Number of calls
├── External API cost
└── Compute cost

RAG
├── Embedding calls
├── Vector queries
├── Reranking
└── Storage / retrieval cost
```

---

## Per-Request Cost

Every agent execution should have a cost record.

```text id="5j7k2s"
request_id
tenant_id
agent_id
model
input_tokens
output_tokens
llm_cost
tool_cost
retrieval_cost
total_cost
```

Example:

```json id="7f6v3m"
{
  "request_id": "req_123",
  "agent": "research_agent",
  "llm_cost": 0.12,
  "tool_cost": 0.03,
  "retrieval_cost": 0.01,
  "total_cost": 0.16
}
```

---

## Cost Attribution

Agent systems should attribute cost to the correct entity.

```text id="m2q8vp"
Agent Run
    │
    ├── User
    ├── Tenant
    ├── Agent
    ├── Workflow
    └── Task
```

This enables questions such as:

> Which tenant generates the most cost?

> Which agent is most expensive?

> Which workflow has the highest cost per successful task?

---

## Cost Budgets

Set runtime limits to prevent runaway execution.

```text id="n7r3kx"
Agent
 │
 ▼
Cost = $0.05
 │
 ▼
Cost = $0.10
 │
 ▼
Cost = $0.20
 │
 ▼
Budget = $0.25
 │
 ▼
STOP
```

Common controls include:

```text
MAX_LLM_CALLS
MAX_TOKENS
MAX_TOOL_CALLS
MAX_COST
MAX_ITERATIONS
```

When a budget is reached, the system can:

* Stop execution
* Return partial results
* Switch to a cheaper model
* Ask for approval
* Use a fallback strategy

---

## Cost Optimization

Cost tracking should identify the source of unnecessary spending.

For example:

```text id="a4c7md"
Research Agent

LLM Calls:        12
Tool Calls:        8
Average Tokens:  18k
Total Cost:     $0.82
```

Tracing may reveal:

```text id="7pj5fz"
Problem
→ Agent repeatedly calls the same search tool

Optimization
→ Cache search result
→ Reduce repeated LLM calls
→ Limit iterations
```

Cost optimization should be based on observed behavior rather than simply choosing the cheapest model.

---

## LLM Cost Example

A simplified cost calculation:

```python id="x3m8qc"
def calculate_cost(
    input_tokens,
    output_tokens,
    input_price,
    output_price
):
    return (
        input_tokens / 1_000_000 * input_price
        + output_tokens / 1_000_000 * output_price
    )
```

For example:

```text id="r5k1vz"
Input tokens  → 100,000
Output tokens → 20,000

Cost =
input_tokens × input_price
+
output_tokens × output_price
```

In production, model pricing should be maintained in configuration rather than hardcoded.

---

## Cost + Tracing

Cost becomes much more useful when attached to trace spans.

```text id="z8p4mx"
Trace: request_123
│
├── LLM Span
│   ├── tokens
│   └── cost = $0.03
│
├── Tool Span
│   └── cost = $0.01
│
├── LLM Span
│   ├── tokens
│   └── cost = $0.05
│
└── Total
    └── $0.09
```

Now engineers can identify **which operation caused the cost**.

---

## Production Considerations

Track:

* Token usage
* Model
* Number of LLM calls
* Tool calls
* Retrieval operations
* External API cost
* Compute cost
* Cost per request
* Cost per task
* Cost per tenant
* Cost per agent
* Cost per successful outcome

Also consider:

* Budget enforcement
* Cost alerts
* Anomaly detection
* Caching
* Model routing
* Token limits

---

## When to Use

Cost tracking is essential when:

* Agents use multiple LLM calls
* Execution is long-running
* Multiple models are used
* External APIs have usage charges
* The platform is multi-tenant
* Agent scale is increasing

---

## Trade-offs

| Benefit                    | Cost                  |
| -------------------------- | --------------------- |
| Cost visibility            | Instrumentation       |
| Budget enforcement         | Additional state      |
| Optimization opportunities | Metadata storage      |
| Tenant-level billing       | Accounting complexity |

---

## Mental Model

> **Don't only measure whether an agent succeeded. Measure what it cost to succeed.**

```text id="j3m7qp"
Agent Execution
      ↓
LLM + Tools + RAG
      ↓
Cost Attribution
      ↓
Request / Agent / Tenant
      ↓
Budget + Optimization
```
