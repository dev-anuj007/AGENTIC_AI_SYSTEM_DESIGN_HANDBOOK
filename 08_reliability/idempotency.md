# Idempotency Pattern

## Problem

Agentic systems frequently retry operations because of:

* Network failures
* Timeouts
* Agent failures
* Duplicate messages
* Workflow recovery

For read operations, repeating a request is usually harmless.

For write operations, repeating the same request can create duplicate side effects.

```text id="k8m2qx"
Agent
  │
  ▼
Payment API
  │
  ▼
Payment succeeds
  │
  X ← response lost
  │
  ▼
Retry
  │
  ▼
Second payment ❌
```

**Idempotency** ensures that repeating the same logical operation produces the same effective result.

---

## Core Idea

Give each logical operation a unique **idempotency key**.

```text id="q4v7mp"
Agent
  │
  │ request_id = "payment-123"
  ▼
Payment Service
  │
  ├── First request → Execute
  │
  └── Duplicate request → Return stored result
```

The service records the key and associates it with the operation's result.

---

## Example: Agent Processing a Refund

Suppose an agent decides to refund an order.

```text id="x6n3vz"
Agent
  │
  │ refund(order=123, key="refund-123")
  ▼
Payment Service
  │
  ├── key not found
  │      ↓
  │   Process refund
  │      ↓
  │   Store result
  │
  └── key already exists
         ↓
      Return previous result
```

If the agent retries after a timeout:

```text id="m5p8qw"
Retry
  │
  ▼
key = "refund-123"
  │
  ▼
Already processed
  │
  ▼
Return original refund result
```

The customer is not refunded twice.

---

## Implementation

A simplified service-side implementation:

```python
def process_refund(order_id, amount, idempotency_key):

    existing = idempotency_store.get(idempotency_key)

    if existing:
        return existing

    result = payment_provider.refund(
        order_id=order_id,
        amount=amount
    )

    idempotency_store.save(
        idempotency_key,
        result
    )

    return result
```

In production, the **check + operation + record** must be designed carefully to avoid race conditions.

---

## Idempotency vs Deduplication

They are related but not identical.

```text id="r3k9vx"
Deduplication
→ Detect duplicate messages/requests

Idempotency
→ Make repeated execution produce
  the same effective outcome
```

A robust system may use both.

---

## Where to Generate the Key

The key should represent the **logical operation**, not the individual network attempt.

```text id="p7m4nc"
Logical operation
      │
      ▼
idempotency_key = "refund-order-123"
      │
      ├── Attempt 1
      ├── Attempt 2
      └── Attempt 3
```

All retries use the same key.

Generating a new key for every retry defeats idempotency.

---

## When to Use

Use idempotency for:

* Payments
* Refunds
* Orders
* Email/SMS sending
* Database writes
* Resource creation
* Agent-to-agent commands
* Workflow retries
* Message processing

It is especially important when an operation has **external side effects**.

---

## When NOT to Use

It is usually unnecessary for naturally idempotent operations such as:

```text id="v5q8mx"
GET /user/123
GET /order/456
Read database record
Search documents
```

Although duplicate reads may still need rate limiting or caching.

---

## Production Considerations

A production implementation should define:

* Idempotency key format
* Key scope
* TTL/retention
* Result storage
* Concurrent request handling
* Conflict behavior
* Security of keys
* Transaction boundaries

A particularly important race condition is:

```text id="z2n7kp"
Request A ──┐
            ├── Check key → not found
Request B ──┘
            └── Check key → not found

Both execute the operation ❌
```

Use atomic operations, unique constraints, locks, or transactional mechanisms to prevent this.

---

## Trade-offs

| Benefit                          | Cost                                 |
| -------------------------------- | ------------------------------------ |
| Prevents duplicate side effects  | Requires state storage               |
| Makes retries safer              | Key lifecycle management             |
| Improves distributed reliability | Additional implementation complexity |
| Useful for workflow recovery     | Storage/cleanup overhead             |

---

## Mental Model

> **Idempotency = the same logical request can safely be executed multiple times without producing duplicate effects.**

The key relationship is:

```text id="e6r3vq"
Retry
  ↓
Same Idempotency Key
  ↓
Same Logical Operation
  ↓
One Effective Side Effect
```
