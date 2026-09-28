# Checkpoint and Resume Pattern

## Problem

Agentic workflows can be long-running and may contain many expensive steps.

If execution fails halfway through, restarting from the beginning wastes:

* Time
* Tokens
* Tool calls
* Compute
* External API calls

The **Checkpoint and Resume Pattern** periodically saves execution state so the workflow can continue from a known point.

---

## Core Idea

```text id="x7m3qp"
Start
  │
  ▼
Step A
  │
  ▼
Checkpoint
  │
  ▼
Step B
  │
  X ← Failure
  │
  ▼
Load Checkpoint
  │
  ▼
Resume Step B
  │
  ▼
Step C
  │
  ▼
Complete
```

The checkpoint represents a consistent state from which execution can safely continue.

---

## Example: AI Research Workflow

Suppose an agent researches a company across several sources:

```text id="p4n8vx"
Research Company
      │
      ├── Company Data ✓
      │
      ├── Financial Data ✓
      │
      ├── Security Data ✓
      │
      ├── Market Research ← failure
      │
      └── Final Report
```

After each major stage, the system saves a checkpoint.

If market research fails:

```text id="m6q2zr"
Load Checkpoint
      │
      ▼
Security Data already complete
      │
      ▼
Resume Market Research
      │
      ▼
Final Report
```

The completed work does not need to be repeated.

---

## Checkpoint Contents

A checkpoint should contain the minimum state required to resume safely.

```text id="q9v4mk"
task_id
workflow_state
current_step
completed_steps
tool_results
agent_outputs
pending_actions
retry_count
timestamp
```

Avoid persisting unnecessary intermediate data.

---

## Checkpoint vs Durable State

They are closely related but serve different purposes.

```text id="w3n7px"
Durable State
→ Persistent execution state

Checkpoint
→ A consistent snapshot of that state
   from which execution can resume
```

Durable state is the broader concept.

Checkpointing is the mechanism used to create recovery points.

---

## LangGraph Implementation

LangGraph supports checkpointing through a **checkpointer**. The graph state is persisted at execution boundaries, allowing an interrupted execution to be resumed using the same thread.

```python
from typing import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    company: str
    company_data: str
    financial_data: str
    market_data: str


def collect_company_data(state: State):
    return {
        "company_data": f"Company data for {state['company']}"
    }


def collect_financial_data(state: State):
    return {
        "financial_data": f"Financial data for {state['company']}"
    }


def collect_market_data(state: State):
    # Imagine this dependency temporarily fails.
    return {
        "market_data": f"Market data for {state['company']}"
    }


def generate_report(state: State):
    return {
        "market_data": (
            state["market_data"]
            + " → Final report generated"
        )
    }


graph = StateGraph(State)

graph.add_node("company", collect_company_data)
graph.add_node("financial", collect_financial_data)
graph.add_node("market", collect_market_data)
graph.add_node("report", generate_report)

graph.add_edge(START, "company")
graph.add_edge("company", "financial")
graph.add_edge("financial", "market")
graph.add_edge("market", "report")
graph.add_edge("report", END)

checkpointer = InMemorySaver()

app = graph.compile(checkpointer=checkpointer)
```

Execution uses a **thread ID** to identify the workflow instance:

```python
config = {
    "configurable": {
        "thread_id": "research-123"
    }
}

result = app.invoke(
    {
        "company": "ACME",
        "company_data": "",
        "financial_data": "",
        "market_data": "",
    },
    config=config
)
```

The important part is:

```text
thread_id = "research-123"
```

The checkpointer associates the execution state with that thread.

If execution is interrupted, the same thread can be used to continue the workflow:

```python
result = app.invoke(
    None,
    config=config
)
```

Conceptually:

```text
First Execution
      │
      ├── Company Data ✓
      │
      ├── Financial Data ✓
      │
      X── Market Data fails
      │
      ▼
Checkpoint State
      │
      │  process restarts
      ▼
Resume thread_id="research-123"
      │
      ▼
Market Data
      │
      ▼
Final Report
```

For production systems, replace `InMemorySaver` with a **durable checkpointer/storage backend** so state survives process or machine failure.

---

## When to Checkpoint

Useful checkpoint boundaries include:

* After expensive operations
* After successful external calls
* Before side effects
* After major workflow stages
* Before human approval
* At safe recovery points

Do not checkpoint after every tiny operation unless the recovery value justifies the overhead.

---

## Failure Recovery

A recovery flow typically looks like:

```text id="h5p8qc"
Execution
   │
   X Failure
   │
   ▼
Find Latest Checkpoint
   │
   ▼
Validate Checkpoint
   │
   ▼
Restore State
   │
   ▼
Resume
```

The system should verify that the checkpoint is valid and compatible with the current workflow version.

---

## Idempotency

Checkpointing should be combined with idempotency.

Consider:

```text id="s7k2mv"
External API succeeds
       │
       X
Process crashes before checkpoint
       │
       ▼
Resume
       │
       ▼
External API called again
```

If the API performs a side effect, the operation may execute twice.

Use idempotency keys for such operations.

---

## When to Use

Use checkpoint/resume when:

* Workflows are long-running
* Tasks are expensive
* Execution can be interrupted
* Human approval pauses execution
* External dependencies can fail
* Work needs reliable recovery

---

## When NOT to Use

It may be unnecessary when:

* The workflow is short
* Recomputing is cheap
* There are no meaningful recovery points
* Execution is intentionally ephemeral

---

## Production Considerations

A production system should handle:

* Atomic checkpoint writes
* State versioning
* Concurrent executions
* Checkpoint corruption
* Retention/cleanup
* Encryption and access control
* Workflow version compatibility
* Idempotent side effects

A checkpoint should represent a **consistent state**, not a partially completed operation.

---

## Trade-offs

| Benefit                        | Cost                        |
| ------------------------------ | --------------------------- |
| Avoids recomputation           | Storage overhead            |
| Supports crash recovery        | State management complexity |
| Enables long-running workflows | Checkpoint coordination     |
| Supports human-in-the-loop     | Versioning requirements     |

---

## Mental Model

> **Checkpoint = "Save enough state so I can safely continue from here if execution stops."**

The key relationship is:

```text id="e4r8ny"
Execute
  ↓
Checkpoint
  ↓
Execute
  ↓
Failure
  ↓
Restore Checkpoint
  ↓
Resume
```
