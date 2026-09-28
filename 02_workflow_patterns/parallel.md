# Parallel Workflow

## 1. Problem

A sequential workflow executes steps one after another:

```text
A → B → C
```

But if multiple tasks are independent, waiting for each one unnecessarily increases latency.

Example:

```text
                 ┌→ User Profile ──┐
Request ─────────┼→ Order History ──┼→ Aggregate
                 └→ Payment History ┘
```

The branches can execute concurrently, so total latency can approach the slowest branch rather than the sum of all branches.

---

## 2. Core Idea

A **Parallel Workflow** executes independent tasks concurrently.

The key question is:

> **Does this task depend on the output of another task?**

If yes:

```text
A → B
```

If no:

```text
A
B
C
```

they are candidates for parallel execution.

The developer still defines the workflow structure; parallelism changes **when independent work executes**, not who decides what happens next.

---

## 3. Dependency Graph

Think about dependencies rather than simply drawing a sequence.

```text
A ──────┐
        ├→ C
B ──────┘
```

Execution:

```text
A ──────────┐
            ├→ C
B ──────┐   │
        └───┘
```

Therefore:

* `A` and `B` can run concurrently.
* `C` must wait for both.

This dependency-graph mindset is the most important skill when identifying parallelism.

---

## 4. Fan-Out / Fan-In

Parallel workflows commonly have two stages.

### Fan-Out

One input produces multiple independent tasks:

```text
             Input
               │
        ┌──────┼──────┐
        ↓      ↓      ↓
        A      B      C
```

### Fan-In

Results are collected before continuing:

```text
        A ─────┐
        B ─────┼→ Aggregate
        C ─────┘
```

Combined:

```text
Input
  ↓
Fan-Out
  ↓
A   B   C
 \  |  /
  Fan-In
     ↓
  Continue
```

This is particularly useful for parallel retrieval, independent API calls, and specialized agents.

---

## 5. Example: Enterprise Research

Suppose an AI system needs to analyze a company using:

* Internal documents
* Web research
* Financial data
* Security data

These sources are independent.

```text
                       Company
                          │
        ┌─────────┬───────┼───────┬─────────┐
        ↓         ↓       ↓       ↓
     Internal    Web   Financial Security
       RAG      Search    Data      Data
        │         │       │         │
        └─────────┴───────┴─────────┘
                          ↓
                      Synthesis
                          ↓
                        Report
```

The synthesis step cannot begin until the required results are available.

---

## 6. Parallelism vs Agentic Reasoning

Parallelism and agent autonomy solve different problems.

### Parallelism

Answers:

> How can independent work execute concurrently?

### Agent

Answers:

> What should the system do next?

They can be combined:

```text
                 Coordinator
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
       Research    Security     Finance
        Agent       Agent        Agent
          │           │           │
          └───────────┼───────────┘
                      ↓
                  Synthesis
```

The coordinator may determine the high-level plan, while independent agents execute their assigned tasks concurrently.

---

## 7. Error Handling

Parallel execution requires an explicit failure policy.

Suppose:

```text
A ✓
B ✗
C ✓
```

Possible strategies:

### Fail Entire Workflow

Use when every result is mandatory.

```text
A ✓
B ✗ → Workflow fails
C ✓
```

### Continue With Partial Results

Use when degraded output is acceptable.

```text
A ✓
B ✗
C ✓
 ↓
Partial Results
```

### Retry

Retry only the failed branch:

```text
A ✓
B ✗ → Retry B
C ✓
```

### Fallback

Replace the failed source:

```text
B
↓
Failure
↓
Fallback B'
```

The correct policy depends on business requirements.

---

## 8. Concurrency Limits

Parallel does not mean unlimited concurrency.

For example:

```text
10,000 documents
       ↓
Concurrency = 100
       ↓
100 tasks running
       ↓
Next 100
       ↓
...
```

Without limits, concurrency can overwhelm:

* APIs
* Databases
* LLM providers
* Worker pools
* Network connections
* Memory

Production workflow systems therefore commonly provide concurrency controls.

---

## 9. Shared State

Avoid unnecessary shared mutable state between branches.

Prefer:

```text
A → result_a ─┐
B → result_b ─┼→ Aggregator
C → result_c ─┘
```

over:

```text
A ─┐
B ─┼→ shared_state
C ─┘
```

Independent outputs make race conditions and debugging easier to manage.

If shared state is unavoidable, define ownership and synchronization explicitly.

---

## 10. Cost and Latency

Parallelism primarily changes **latency**, not the amount of work.

Suppose three LLM calls each take two seconds:

Sequential:

```text
2 + 2 + 2 = ~6 sec
```

Parallel:

```text
max(2, 2, 2) = ~2 sec
```

The token usage is still approximately the same.

However, concurrency can increase:

* Instantaneous resource usage
* Provider rate-limit pressure
* Peak cost
* Infrastructure load

So optimize for latency only when the additional concurrency is acceptable.

---

## 11. When to Use

Use parallel execution when:

* Tasks are independent.
* Latency matters.
* Multiple data sources can be queried independently.
* Multiple agents work on separate subtasks.
* Large collections contain independently processable items.

Typical examples:

```text
Parallel API calls
Parallel RAG retrieval
Parallel document processing
Parallel agent execution
Parallel LLM evaluations
```

---

## 12. When Not to Use

Avoid parallelism when:

### Strong dependency exists

```text
A → B → C
```

### Ordering matters

```text
Create Account
    ↓
Create Payment Method
    ↓
Create Subscription
```

### Shared state creates race conditions

### Downstream systems cannot handle the concurrency

### The performance improvement is insignificant compared with the added complexity

---

## 13. Production Checklist

Before introducing parallel execution:

```text
□ Are the branches independent?

□ What inputs does each branch require?

□ What outputs does each branch produce?

□ Do we need every result?

□ What happens when one branch fails?

□ Can failed branches be retried?

□ Can partial results be accepted?

□ What is the concurrency limit?

□ Can downstream services handle the load?

□ Are there rate limits?

□ Is shared mutable state required?

□ How are results aggregated?

□ How are individual branches traced?
```

---

## 14. Interview Example

### Question

> "How would you reduce latency in an AI research system that queries multiple independent sources?"

Answer:

> "I would model the workflow as a dependency graph. Independent retrieval operations can be fanned out and executed concurrently, then their results can be fanned in to a synthesis step. I would also define concurrency limits, retry and partial-failure policies, and avoid shared mutable state between branches."

Architecture:

```text
                 User Query
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
        Web        Internal   Financial
       Search        RAG        Data
          │          │          │
          └──────────┼──────────┘
                     ↓
                 Synthesis
```

---

## 15. Trade-offs

| Aspect           | Parallel Workflow              |
| ---------------- | ------------------------------ |
| Execution        | Concurrent                     |
| Dependency       | Branches should be independent |
| Latency          | Lower for independent work     |
| Resource usage   | Higher                         |
| Complexity       | Higher than sequential         |
| Debugging        | More complex                   |
| Failure handling | Requires explicit policy       |
| Best for         | Independent tasks              |

The fundamental trade-off:

```text
More concurrency
      ↓
Lower latency
      ↓
Higher resource usage + complexity
```

---

## 16. Mental Model

```text
Sequential
A → B → C

Parallel
A ─┐
B ─┼→ Merge
C ─┘
```

Remember:

> **Parallelize independent work and synchronize at dependency boundaries.**

That is the core pattern.

---

## 17. Relationship to Other Patterns

```text
Sequential
    ↓
Parallel
    ↓
Fan-Out / Fan-In
    ↓
Router
    ↓
Conditional
    ↓
Loop
```

The distinction between the workflow patterns is:

| Pattern        | Main Question                               |
| -------------- | ------------------------------------------- |
| Sequential     | What is the fixed order?                    |
| Parallel       | What independent work can run together?     |
| Fan-Out/Fan-In | How do we split work and combine results?   |
| Router         | Which path should execute?                  |
| Conditional    | Should a step execute based on a condition? |
| Loop           | Should we repeat a step?                    |

Parallel is therefore about **concurrency**, not dynamic decision-making.
