# Tool Calling

## Problem

An LLM can reason about a task, but it cannot directly interact with external systems.

To perform actions such as querying a database, calling an API, sending an email, or executing code, the model needs access to **tools**.

---

## Core Idea

The model decides when a tool is required and provides structured arguments.

```text id="k8m3qv"
User Request
     │
     ▼
    LLM
     │
     │ tool call
     ▼
   Tool
     │
     │ result
     ▼
    LLM
     │
     ▼
Final Response
```

The application remains responsible for executing the actual tool.

---

## Example: Customer Support Agent

A customer asks:

> "Where is my order 12345?"

The agent determines that it needs order information.

```text id="p4n7xz"
User
 │
 ▼
LLM
 │
 └── get_order_status(order_id=12345)
              │
              ▼
        Order Service
              │
              ▼
        status = shipped
              │
              ▼
             LLM
              │
              ▼
     "Your order has shipped."
```

The LLM does not directly access the order database.

It requests the capability through a defined tool interface.

---

## Tool Contract

A tool should have a clear schema:

```json id="c6v2mp"
{
  "name": "get_order_status",
  "description": "Get the current status of an order",
  "input": {
    "order_id": "string"
  }
}
```

The application validates the arguments before executing the operation.

---

## Read vs Write Tools

Tools can have very different risk levels.

### Read

```text
get_customer()
search_documents()
query_database()
```

### Write

```text
refund_payment()
delete_user()
send_email()
deploy_application()
```

Write operations require stronger authorization and often human approval.

---

## Tool Execution Boundary

The model should **request** an action, not receive unrestricted access to infrastructure.

```text id="q7m2nx"
LLM
 │
 ▼
Tool Request
 │
 ▼
Validation
 │
 ├── Authorization
 ├── Input validation
 ├── Policy checks
 └── Rate limits
 │
 ▼
Tool Execution
 │
 ▼
Result
```

This boundary is critical for production systems.

---

## When to Use

Use tool calling when the agent needs to:

* Retrieve external information
* Interact with APIs
* Query databases
* Execute controlled actions
* Perform calculations
* Access enterprise systems

---

## When NOT to Use

Do not create a tool when:

* The model already has the required information
* A deterministic application step is simpler
* The operation does not require model-driven decision-making

Not every function in an application needs to become an LLM tool.

---

## Production Considerations

Production tools should have:

* Strict input schemas
* Authentication and authorization
* Timeouts
* Retry policies
* Rate limits
* Idempotency for side effects
* Audit logs
* Structured errors
* Observability
* Permission boundaries

Tool descriptions should also be precise. Poor descriptions can cause the model to select the wrong tool or generate incorrect arguments.

---

## Tool Failure

Tools can fail independently of the model.

```text id="s9k4wp"
Tool Call
   │
   ├── Success → Continue
   │
   ├── Retryable Failure → Retry
   │
   ├── Invalid Request → Fix / Re-plan
   │
   └── Permanent Failure → Fallback / Escalate
```

The agent should receive structured errors rather than raw infrastructure exceptions whenever possible.

---

## Trade-offs

| Benefit                     | Cost                                    |
| --------------------------- | --------------------------------------- |
| Extends agent capabilities  | Tool execution adds latency             |
| Connects enterprise systems | Requires security controls              |
| Enables real-world actions  | Tool failures become part of the system |
| Supports dynamic behavior   | Model may select incorrect tools        |

---

## Mental Model

> **Tool calling gives an agent controlled capabilities to interact with the outside world.**

The important boundary is:

```text
LLM decides WHAT should happen
        ↓
Application decides WHETHER it is allowed
        ↓
Tool executes HOW it happens
```
