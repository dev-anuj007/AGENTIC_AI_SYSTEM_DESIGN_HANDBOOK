# Tool Evaluation

## Problem

An agent can make a reasonable decision and still fail because the tool it selected or invoked was inappropriate.

For example:

```text
Agent Decision
      ↓
Wrong Tool
      ↓
Wrong / Unsafe Result
```

Tool evaluation checks whether the agent **selected, invoked, and used tools correctly**.

---

## Core Idea

Evaluate tool usage independently from the final answer and overall trajectory.

```text id="u8k4pz"
Agent Decision
      │
      ▼
Tool Selection
      │
      ▼
Tool Arguments
      │
      ▼
Tool Execution
      │
      ▼
Tool Result
      │
      ▼
Tool Evaluator
```

The goal is to determine:

> **Was the right tool used correctly for the intended task?**

---

## Example: Customer Support Agent

Suppose a customer asks:

> "Where is my order?"

The agent has:

```text id="j3q7mx"
get_order_status
cancel_order
refund_order
update_address
```

A correct tool trajectory is:

```text
User Request
     ↓
get_order_status
     ↓
order status
```

Calling `refund_order` would be an incorrect tool choice even if the final response looks reasonable.

---

## What to Evaluate

### 1. Tool Selection

Was the correct tool selected?

```text
Order status → get_order_status ✓
Order status → refund_order    ✗
```

### 2. Arguments

Were the arguments correct?

```json id="m5r8qa"
{
  "order_id": "ORD-123"
}
```

Check:

* Required fields
* Types
* Valid values
* Resource scope
* User/tenant context

### 3. Permission

Was the agent authorized to use the tool?

```text
Agent
  ↓
Tool Request
  ↓
Authorization
  ├── Allowed → Execute
  └── Denied  → Reject
```

Tool evaluation should not replace the authorization system.

### 4. Result Handling

Did the agent correctly interpret the tool result?

A tool may return:

```json id="p9v2kx"
{
  "status": "failed",
  "error": "ORDER_NOT_FOUND"
}
```

The agent should not interpret this as a successful order lookup.

---

## Rule-Based Evaluation

Many tool properties can be evaluated deterministically.

```python id="x6m3qa"
def evaluate_tool_call(call):
    if call["tool"] not in ALLOWED_TOOLS:
        return "invalid_tool"

    if not validate_schema(call["arguments"]):
        return "invalid_arguments"

    if not authorized(call):
        return "unauthorized"

    return "valid"
```

This is useful for:

* Schema validation
* Required parameters
* Allowed tools
* Permission checks
* Resource boundaries
* Rate limits

---

## LLM-Based Evaluation

Semantic questions may require an evaluator model.

For example:

> Given the user request and available tools, was this tool the most appropriate choice?

Input:

```text id="a2k7vx"
User Goal
+
Available Tools
+
Selected Tool
+
Arguments
+
Tool Result
```

Output:

```json id="r8m4qp"
{
  "decision": "pass",
  "reason": "Tool matches the user's request",
  "argument_quality": "valid"
}
```

For high-risk operations, deterministic policy checks should remain authoritative.

---

## Tool Evaluation vs Trajectory Evaluation

They overlap, but their scope is different.

| Tool Evaluation                   | Trajectory Evaluation              |
| --------------------------------- | ---------------------------------- |
| Focuses on individual tool usage  | Focuses on complete execution path |
| Was the right tool selected?      | Was the overall strategy good?     |
| Were arguments valid?             | Were decisions well sequenced?     |
| Was result interpreted correctly? | Was execution efficient?           |

Example:

```text id="q7m3nz"
Trajectory
   │
   ├── Search Tool ✓
   ├── Database Tool ✓
   ├── Search Tool ✓
   └── Final Answer
```

Trajectory evaluation asks whether this overall path was appropriate.

Tool evaluation asks whether each individual tool interaction was correct.

---

## Production Metrics

Useful metrics include:

* Tool selection accuracy
* Invalid tool-call rate
* Invalid argument rate
* Authorization denial rate
* Tool failure rate
* Tool retry rate
* Unnecessary tool-call rate
* Tool latency
* Tool cost
* Result interpretation errors

Track these by:

```text
Agent
Model
Tool
Workflow
Version
Tenant
```

This helps identify whether a problem comes from the model, tool contract, orchestration, or tool implementation.

---

## When to Use

Tool evaluation is especially useful for:

* Tool-heavy agents
* MCP-based systems
* Multi-agent platforms
* Agents with many available tools
* High-risk tools
* Production agents
* Tool-selection optimization

---

## When NOT to Use

A separate tool evaluator may be unnecessary when:

* There are very few tools
* Tool selection is deterministic
* Tools have strict schemas and simple routing
* The workflow does not allow meaningful tool choice

---

## Trade-offs

| Benefit                      | Cost                    |
| ---------------------------- | ----------------------- |
| Detects incorrect tool usage | Additional evaluation   |
| Improves tool selection      | Telemetry/storage       |
| Finds argument errors        | Engineering complexity  |
| Helps optimize agents        | Possible evaluator cost |

---

## Mental Model

> **Tool evaluation asks whether the agent used the right capability, with the right inputs, in the right way.**

```text id="v5q8mx"
Right Goal
   ↓
Right Tool
   ↓
Right Arguments
   ↓
Right Permission
   ↓
Right Result Handling
```
