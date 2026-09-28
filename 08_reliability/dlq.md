# Dead Letter Queue (DLQ) Pattern

## Problem

Some tasks continue failing even after retries.

If failed work keeps retrying indefinitely, it can:

* Consume resources
* Delay other work
* Increase cost
* Create retry storms
* Hide permanent failures

A **Dead Letter Queue (DLQ)** stores messages or tasks that could not be successfully processed after the allowed retry attempts.

---

## Core Idea

```text id="d9k4mx"
Request
   │
   ▼
Queue
   │
   ▼
Worker
   │
   ├── Success ─────────→ Complete
   │
   └── Failure
          │
          ▼
        Retry
          │
          ├── Success → Complete
          │
          └── Max Retries
                 │
                 ▼
                DLQ
                 │
                 ▼
           Investigate / Replay
```

The DLQ is not the recovery mechanism itself.

It is a **controlled holding area for work that needs investigation or later recovery**.

---

## Example: AI Notification Agent

Suppose an agent processes notification requests:

```text id="q2v8ps"
Notification Request
        │
        ▼
       SQS
        │
        ▼
Notification Worker
        │
        ├── Email ✓
        ├── SMS ✓
        └── Slack ✗
                │
                ▼
              Retry
                │
                X
          Max retries
                │
                ▼
               DLQ
```

The DLQ message can contain:

```text id="m7x3qa"
request_id
notification_type
recipient
payload
failure_reason
retry_count
timestamp
correlation_id
```

An operator or automated recovery process can later inspect and replay the failed request.

---

## Retry vs DLQ

They solve different problems.

| Retry                             | DLQ                                           |
| --------------------------------- | --------------------------------------------- |
| Handles temporary failures        | Handles repeatedly failed work                |
| Attempts the same operation again | Stores failed work for investigation/recovery |
| Short-term recovery               | Longer-term failure handling                  |
| Usually automatic                 | Often requires controlled replay              |

Typical flow:

```text
Failure
  ↓
Retry
  ↓
Retry
  ↓
Retry
  ↓
DLQ
```

---

## What Should Go to the DLQ?

Common candidates:

* Repeated dependency failures
* Invalid payloads
* Schema mismatches
* Poison messages
* Permanent business-rule failures
* Unexpected processing exceptions

Not every failure should be retried.

For example:

```text
Invalid Request
      ↓
Do not retry blindly
      ↓
DLQ / Error Queue
```

Whereas:

```text
Temporary Network Failure
      ↓
Retry
```

---

## Replay

A DLQ becomes useful only if failed work can be safely recovered.

A replay process might look like:

```text id="w6p2nr"
DLQ
 │
 ▼
Inspect Failure
 │
 ├── Permanent → Fix / Discard
 │
 └── Recoverable
        │
        ▼
   Validate / Repair
        │
        ▼
      Replay
        │
        ▼
      Queue
```

Replay should normally use the **original request ID or idempotency key** so the operation does not create duplicate side effects.

---

## Concrete Implementation

A simplified worker:

```python id="z8r4kc"
def process_message(message):

    try:
        handle_request(message)

    except TemporaryError:
        raise  # Queue system retries

    except PermanentError as error:
        send_to_dlq(
            message=message,
            reason=str(error)
        )
```

The queue infrastructure can control:

```text
max_attempts
retry_delay
backoff
DLQ destination
visibility_timeout
```

For AWS SQS, this is commonly implemented using a **redrive policy**:

```text
Main Queue
    │
    ▼
Lambda / Worker
    │
    ├── Success → Delete message
    │
    └── Failure
          │
          ▼
      Visibility timeout
          │
          ▼
        Retry
          │
          ▼
      maxReceiveCount
          │
          ▼
         DLQ
```

---

## DLQ Monitoring

A DLQ should be monitored.

Important metrics include:

* DLQ message count
* Messages entering DLQ
* Oldest message age
* Failure reason
* Retry count
* Replay success rate

A sudden increase can indicate:

```text
Dependency outage
       OR
Bad deployment
       OR
Schema change
       OR
Poison message
       OR
Downstream API failure
```

---

## When to Use

Use a DLQ when:

* Messages are processed asynchronously
* Failures can persist beyond retry attempts
* Failed work needs investigation
* Messages may require replay
* Losing failed work is unacceptable

---

## When NOT to Use

A DLQ may be unnecessary when:

* Work is synchronous
* Failure can simply be returned to the caller
* Failed operations have no recovery value
* The system has no mechanism to inspect or replay failures

---

## Production Considerations

A production DLQ should define:

* Maximum retry attempts
* Retry/backoff policy
* Failure classification
* Message retention
* Monitoring and alerting
* Replay mechanism
* Idempotency
* Access control
* Audit trail
* Poison-message handling

Never automatically replay an entire DLQ without understanding the failure cause.

---

## Trade-offs

| Benefit                      | Cost                      |
| ---------------------------- | ------------------------- |
| Prevents infinite retries    | Additional infrastructure |
| Preserves failed work        | Requires monitoring       |
| Enables investigation        | Replay complexity         |
| Supports controlled recovery | Storage cost              |

---

## Mental Model

> **Retry handles temporary failure. DLQ isolates work that could not be successfully processed so it can be investigated, repaired, or safely replayed.**
