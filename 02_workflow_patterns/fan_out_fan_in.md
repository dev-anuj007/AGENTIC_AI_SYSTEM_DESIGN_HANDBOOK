# Fan-Out / Fan-In Workflow

## 1. Problem

Some tasks can be divided into multiple independent subtasks, but the workflow cannot continue until their results are combined.

Example:

> Analyze a company across financial, security, and market dimensions.

Instead of processing them sequentially:

```text
Company
  ↓
Financial
  ↓
Security
  ↓
Market
  ↓
Report
```

we can execute the independent analyses concurrently and then combine their results:

```text
                  Company
                     │
                 Fan-Out
          ┌──────────┼──────────┐
          ↓          ↓          ↓
      Financial   Security    Market
          │          │          │
          └──────────┼──────────┘
                   Fan-In
                     ↓
                  Report
```

---

## 2. Core Idea

**Fan-Out** distributes work into multiple independent branches.

**Fan-In** waits for those branches and combines their results.

```text
Input
  ↓
Fan-Out
  ↓
A   B   C
  ↓ ↓ ↓
Fan-In
  ↓
Output
```

The key property is:

> **Branches can execute independently, but the next stage depends on their combined results.**

---

## 3. Example: Company Intelligence System

A company-analysis system receives:

```text
Company = ACME
```

The workflow creates three tasks:

```text
Financial Agent
Security Agent
Market Agent
```

Each produces a structured result:

```text
financial_result
security_result
market_result
```

The aggregator receives all three:

```text
{
    "financial": financial_result,
    "security": security_result,
    "market": market_result
}
```

and generates the final report.

Architecture:

```text
                       ACME
                        │
                     Fan-Out
              ┌─────────┼─────────┐
              ↓         ↓         ↓
          Financial   Security   Market
            Agent      Agent      Agent
              │         │         │
              └─────────┼─────────┘
                        ↓
                     Fan-In
                        ↓
                   Report Agent
```

---

## 4. Fan-Out

The system determines the independent units of work.

```text
Company
   ↓
┌─────────────┬─────────────┬─────────────┐
↓             ↓             ↓
Financial     Security      Market
```

Each branch should ideally:

* Have clearly defined inputs
* Produce an independent result
* Avoid modifying shared mutable state

This makes the branches independently executable and retryable.

---

## 5. Fan-In

Fan-in synchronizes the branches.

```text
Financial ──┐
Security  ──┼→ Aggregator
Market    ──┘
```

The aggregator can then:

* Combine results
* Remove duplicates
* Resolve conflicts
* Calculate derived values
* Ask an LLM to synthesize the final result

Fan-in therefore acts as a **dependency barrier**:

> The downstream step starts only after the required upstream branches have completed.

---

## 6. Handling Failures

Suppose:

```text
Financial ✓
Security  ✗
Market    ✓
```

The system needs an explicit policy.

### All Results Required

```text
Security fails
     ↓
Retry
     ↓
Still failing
     ↓
Workflow fails
```

### Partial Results Allowed

```text
Financial ✓
Security  ✗
Market    ✓
     ↓
Partial Results
     ↓
Report with missing security analysis
```

The policy depends on whether each branch is mandatory or optional.

---

## 7. Fan-Out Granularity

Fan-out can happen at different scales.

### Small Fixed Set

```text
Company
 ├── Financial
 ├── Security
 └── Market
```

### Large Dynamic Set

```text
10,000 Documents
      ↓
Fan-Out
      ↓
Document 1
Document 2
...
Document 10,000
```

For large workloads, do not create unlimited concurrent tasks.

Use:

```text
10,000 tasks
    ↓
Concurrency limit = 100
```

This protects downstream systems and controls resource usage.

---

## 8. When to Use

Use Fan-Out / Fan-In when:

* A problem naturally decomposes into independent subtasks.
* The subtasks can execute concurrently.
* A later stage needs their combined results.
* You need parallel processing with explicit synchronization.

Common examples:

```text
Multi-source research
Parallel document processing
Multi-agent analysis
Batch processing
Parallel evaluations
```

---

## 9. When Not to Use

Do not use it when:

### Tasks are sequentially dependent

```text
A → B → C
```

### There is no aggregation step

If results do not need to come together, simple parallel execution may be enough.

### The branches are too tightly coupled

If branches constantly communicate or modify shared state, the fan-out architecture loses its simplicity.

---

## 10. Production Considerations

Important design concerns:

```text
Concurrency limits
Retry policy
Timeouts
Partial failure handling
Idempotency
Checkpointing
Result aggregation
Observability
```

For each branch, track at least:

```text
workflow_id
branch_id
status
latency
retry_count
error
```

This makes it possible to identify which branch caused a slow or failed workflow.

---

## 11. Trade-offs

| Aspect           | Fan-Out / Fan-In             |
| ---------------- | ---------------------------- |
| Execution        | Parallel branches            |
| Synchronization  | Required                     |
| Latency          | Lower for independent work   |
| Resource usage   | Higher                       |
| Failure handling | More complex                 |
| Aggregation      | Required                     |
| Best for         | Decomposable multi-step work |

The main trade-off is:

> **More parallelism and lower latency in exchange for concurrency, synchronization, and failure-management complexity.**

---

## 12. Mental Model

Remember:

```text
             Fan-Out
                ↓
          ┌─────┼─────┐
          ↓     ↓     ↓
          A     B     C
          └─────┼─────┘
                ↓
             Fan-In
                ↓
             Result
```

> **Fan-Out divides the work; Fan-In synchronizes and combines the results.**

This pattern is especially useful when independent agents or services analyze different aspects of the same problem.
