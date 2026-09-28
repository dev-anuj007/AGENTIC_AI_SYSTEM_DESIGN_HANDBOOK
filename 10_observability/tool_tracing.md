# Tool Tracing

## Problem

Agentic systems depend heavily on tools.

A single agent execution may call:

```text
Agent
 ├── Search API
 ├── Database
 ├── Web Search
 ├── Payment Service
 └── Notification Service
```

When something goes wrong, you need to know **which tool was called, why, how long it took, and what happened**.

Tool tracing provides visibility into each tool execution.

---

## Core Idea

Every meaningful operation in an agent execution should be represented as a trace span.

The two most important span types are:

```text
LLM Span
→ Model reasoning / generation

Tool Span
→ External action / dependency execution
```

They form one connected trace:

```text
Agent Trace
     │
     ├── LLM Span
     │      └── Model decides to call search_customer
     │
     ├── Tool Span
     │      └── search_customer
     │
     ├── LLM Span
     │      └── Model interprets result
     │
     └── Tool Span
            └── update_customer
```

---

## LLM Span vs Tool Span

| LLM Span                        | Tool Span                        |
| ------------------------------- | -------------------------------- |
| Represents model execution      | Represents tool execution        |
| Captures reasoning/generation   | Captures external action         |
| Model, tokens, latency          | Tool, dependency, latency        |
| Input/output model metadata     | Request/result metadata          |
| Can produce tool calls          | Executes the requested tool      |
| Usually measures inference cost | Usually measures dependency cost |

The key boundary is:

```text
LLM
 │
 │ "Call get_order(order_id=123)"
 ▼
Tool
 │
 │ executes API/database operation
 ▼
Result
 │
 ▼
LLM
```

The LLM span tells you **what the model decided**.

The tool span tells you **what happened when that action was executed**.

---

## Example: Customer Support Agent

A customer asks:

> "Why hasn't my order arrived?"

The agent might execute:

```text
User Request
     │
     ▼
Support Agent
     │
     ├── LLM Span
     │      └── Decide: get_order
     │
     ├── Tool Span
     │      └── get_order()
     │
     ├── LLM Span
     │      └── Decide: get_shipping_status
     │
     ├── Tool Span
     │      └── get_shipping_status()
     │
     ├── LLM Span
     │      └── Decide: create_support_ticket
     │
     └── Tool Span
            └── create_support_ticket()
```

A single trace therefore captures both **decision-making and execution**.

---

## What Each Span Should Capture

### LLM Span

Useful metadata includes:

```text
model
model_version
prompt_version
input_tokens
output_tokens
latency
finish_reason
tool_calls
status
error
```

For example:

```text
LLM Span
────────────────────────
model: gpt-5.6
input_tokens: 1250
output_tokens: 180
latency: 820ms
tool_call: get_order
status: success
```

### Tool Span

Useful metadata includes:

```text
tool_name
tool_version
service
trace_id
span_id
parent_span_id
start_time
duration
status
error_type
retry_count
```

For example:

```text
Tool Span
────────────────────────
tool: get_order
service: order-service
latency: 120ms
status: success
```

Do not blindly record complete prompts, arguments, or responses if they contain sensitive information.

---

## Tool Failure

Suppose the shipping API fails:

```text
LLM Span
  │
  │ Decide: get_shipping_status
  ▼
Tool Span
  │
  X timeout
  │
  ▼
Retry Tool Span
  │
  X timeout
  │
  ▼
LLM Span
  │
  │ Decide: use fallback
  ▼
Fallback Tool Span
```

The trace now shows the difference between:

* Model decision
* Tool execution
* Retry
* Tool failure
* Fallback decision

This is much more useful than a single:

```text
Request failed
```

---

## Tool Tracing with Distributed Systems

A tool often calls another service.

```text
Agent
 │
 │ trace_id = abc123
 ▼
Tool Gateway
 │
 │ trace_id = abc123
 ▼
Shipping Service
 │
 │ trace_id = abc123
 ▼
Database
```

Propagating the same trace context allows the entire operation to appear as one distributed trace.

---

## OpenTelemetry Example

A tool wrapper can create a span around the actual tool call:

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)


def get_shipping_status(tracking_id):

    with tracer.start_as_current_span(
        "tool.get_shipping_status"
    ) as span:

        span.set_attribute(
            "tool.name",
            "get_shipping_status"
        )

        try:
            result = shipping_api.get_status(
                tracking_id
            )

            span.set_attribute(
                "tool.status",
                "success"
            )

            return result

        except Exception as error:

            span.record_exception(error)
            span.set_attribute(
                "tool.status",
                "error"
            )

            raise
```

The resulting trace can look like:

```text
Agent Run
   │
   ├── LLM: decision
   │
   ├── Tool: get_shipping_status
   │      ├── duration
   │      ├── status
   │      └── exception
   │
   └── LLM: final response
```

---

## What Tool Tracing Helps Answer

Production teams should be able to answer:

### Reliability

> Which tools are failing?

### Latency

> Which dependency is slowing down the agent?

### Cost

> Which tools are called most frequently?

### Agent Behavior

> Why did the agent call this tool?

### Security

> Which agent/user invoked this tool?

### Debugging

> What happened immediately before the failure?

---

## Tool Tracing vs Agent Tracing

They operate at different levels.

| Agent Tracing           | Tool Tracing              |
| ----------------------- | ------------------------- |
| Entire agent execution  | Individual tool execution |
| Includes LLM spans      | Includes tool spans       |
| Agent iterations        | Tool attempts             |
| Workflow path           | Dependency behavior       |
| Overall latency         | Tool latency              |
| Model/token information | Service/tool information  |

They should normally be connected through the **same trace**.

---

## Production Considerations

Capture:

* Trace and span IDs
* Parent-child relationships
* LLM spans
* Tool spans
* Start/end time
* Duration
* Status
* Error type
* Retry count
* Timeout
* Service/dependency
* Tenant/user identity where appropriate

Protect sensitive information:

```text
❌ API keys
❌ Passwords
❌ Access tokens
❌ Raw sensitive payloads

✓ Tool name
✓ Request ID
✓ Resource ID where safe
✓ Status
✓ Latency
✓ Error type
```

Use redaction or hashing where appropriate.

---

## When to Use

Tool tracing is especially important when:

* Agents use many tools
* Tools call external services
* Distributed systems are involved
* Tool failures are common
* Latency matters
* Security auditing is required

---

## Trade-offs

| Benefit               | Cost                    |
| --------------------- | ----------------------- |
| Tool-level debugging  | Instrumentation         |
| Dependency visibility | Storage                 |
| Latency analysis      | Trace volume            |
| Security auditing     | Sensitive-data handling |

---

## Mental Model

> **LLM spans show what the agent decided. Tool spans show what happened when that decision crossed into the external world.**

```text
                    Agent Trace
                         │
             ┌───────────┴───────────┐
             │                       │
         LLM Span                Tool Span
             │                       │
       "What should I do?"      "Execute it"
             │                       │
             └───────────┬───────────┘
                         │
                    Next LLM Span
```
