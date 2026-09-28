# Retry Pattern

## Problem

Agentic systems depend on components that can fail temporarily:

* LLM APIs
* Tool APIs
* Databases
* Network services
* External agents

A temporary failure should not always cause the entire execution to fail.

The **Retry Pattern** allows an operation to be attempted again according to a controlled policy.

---

## Core Idea

```text id="k7m3qx"
Operation
    │
    ▼
  Execute
    │
 ┌──┴──────────────┐
 │                 │
Success          Failure
 │                 │
 ▼                 ▼
Continue       Is Retryable?
                  │
             ┌────┴────┐
            Yes        No
             │          │
          Backoff      Fail
             │
          Retry
             │
             ▼
         Max Attempts?
```

---

## Example: Agent Calling a Payment Service

An agent needs to retrieve payment status.

```text id="p5v8nc"
Agent
  │
  ▼
Payment API
  │
  ├── Timeout
  │
  ▼
Retry #1
  │
  ├── Timeout
  │
  ▼
Retry #2
  │
  ├── Success
  │
  ▼
Continue Agent Execution
```

The agent should not immediately fail because of a transient network problem.

---

## Retryable vs Non-Retryable

Not every error should be retried.

### Usually Retryable

```text id="q8n2mv"
Timeout
Connection reset
Temporary service unavailable
HTTP 429
HTTP 503
```

### Usually Not Retryable

```text id="r4k7px"
Invalid input
Unauthorized
Forbidden
Malformed request
Business rule violation
```

Retrying a permanent failure only wastes time and money.

---

## Backoff

Retries should not happen immediately.

A common strategy is exponential backoff:

```text id="n6v3qw"
Attempt 1 → wait 1s
Attempt 2 → wait 2s
Attempt 3 → wait 4s
Attempt 4 → stop
```

Add **jitter** so many agents do not retry at exactly the same time.

```text id="w2m8kp"
delay = exponential_backoff + random_jitter
```

---

## Idempotency

Retries become dangerous when an operation has side effects.

For example:

```text id="j5x9mc"
Agent
  ↓
"Send payment"
  ↓
Success
  ↓
Response lost
  ↓
Retry
  ↓
Second payment
```

Use an idempotency key for operations that must not execute twice:

```text id="z3q7nv"
request_id = "payment-12345"
```

The receiving service can recognize duplicate requests.

---

## Retry Budget

Retries should have explicit limits:

```text id="f8m4qx"
MAX_RETRIES = 3
MAX_RETRY_TIME = 30 seconds
MAX_RETRY_COST = ...
```

The system should stop retrying when the budget is exhausted.

---

## When to Use

Use retries when:

* Failures are transient
* The operation is safe to repeat
* The dependency occasionally becomes unavailable
* Recovery is likely without changing the request

---

## When NOT to Use

Avoid retries when:

* The error is deterministic
* The operation is not idempotent
* The dependency is permanently unavailable
* Retrying could amplify an outage
* The operation requires human intervention

---

## Production Considerations

A production retry system should define:

* Retryable error types
* Maximum attempts
* Exponential backoff
* Jitter
* Timeouts
* Idempotency
* Retry budgets
* Logging and metrics
* Circuit-breaker integration

A critical principle:

> **Retries should recover transient failures, not hide systemic failures.**

If a dependency is continuously failing, repeated retries can make the outage worse.

---

## Trade-offs

| Benefit                           | Cost                                  |
| --------------------------------- | ------------------------------------- |
| Recovers transient failures       | Adds latency                          |
| Improves reliability              | Increases cost                        |
| Simple failure recovery           | Can amplify outages                   |
| Useful across distributed systems | Requires idempotency for side effects |

---

## Mental Model

> **Retry = "This operation failed temporarily; try again under a controlled policy."**

The key questions are:

1. **Is the failure transient?**
2. **Is the operation safe to repeat?**
3. **How many retries can we afford?**
4. **When should we stop retrying?**
