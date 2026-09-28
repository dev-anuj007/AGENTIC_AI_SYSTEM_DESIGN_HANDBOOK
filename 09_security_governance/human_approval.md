# Human Approval

## Problem

Some agent actions are too risky to execute autonomously.

Examples include:

* Financial transactions
* Production deployments
* Deleting data
* Sending sensitive information
* Changing permissions
* External communications

The **Human Approval Pattern** introduces a human decision point before executing high-impact actions.

---

## Core Idea

```text
Agent
  │
  ▼
Proposed Action
  │
  ▼
Risk / Policy Check
  │
  ├── Low Risk ─────────→ Execute
  │
  └── High Risk
         │
         ▼
    Human Approval
         │
      ┌──┴──┐
      ▼     ▼
   Approve Reject
      │       │
      ▼       ▼
   Execute   Stop
```

The agent remains autonomous for low-risk actions while humans control high-impact actions.

---

## Example: Production Deployment Agent

A coding agent can:

```text
Read repository      → Automatic
Run tests             → Automatic
Create PR             → Automatic
Deploy staging        → Automatic
Deploy production     → Human approval
```

The agent proposes:

```json
{
  "action": "deploy",
  "environment": "production",
  "version": "v2.4.1",
  "reason": "All tests passed"
}
```

The system presents the action to an authorized human.

```text
Production Deployment
Version: v2.4.1

Tests: Passed
Security Scan: Passed
Environment: Production

[Approve] [Reject]
```

Only after approval does the system execute the deployment.

---

## Approval Boundary

The approval should occur **before the irreversible or high-impact side effect**.

```text
Agent Reasoning
      ↓
Tool Request
      ↓
Policy Check
      ↓
Human Approval  ← control boundary
      ↓
Tool Execution
      ↓
External Side Effect
```

Do not execute the action first and ask for approval afterward.

---

## What Should the Human See?

Approval should contain enough information to make a meaningful decision:

```text
Action
Target
Arguments
Reason
Expected Impact
Risk Level
Agent Identity
User Identity
Relevant Evidence
```

Avoid presenting only:

```text
"Agent wants permission. Approve?"
```

The human needs the context required to understand what will happen.

---

## Approval State

For long-running workflows, approval should be represented as durable state.

```text
approval_id
request_id
requested_by
action
target
status
created_at
expires_at
approved_by
approved_at
```

Possible states:

```text
PENDING
APPROVED
REJECTED
EXPIRED
CANCELLED
```

---

## Approval in an Agent Loop

Human approval can pause an agent loop:

```text
Decide
  ↓
Act?
  ↓
High Risk?
  │
  ├── No → Execute
  │
  └── Yes
       ↓
   Request Approval
       ↓
      PAUSE
       │
       ▼
 Human Decision
       │
   ┌───┴────┐
   ▼        ▼
Approve   Reject
   │        │
   ▼        ▼
Execute    Stop
```

The agent should not continue execution while approval is pending.

---

## LangGraph Implementation

LangGraph can model human approval as an interrupt point.

```python
from typing import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt


class State(TypedDict):
    action: str
    environment: str
    version: str
    approved: bool
    result: str


def check_risk(state: State):
    return state


def request_approval(state: State):
    decision = interrupt({
        "action": state["action"],
        "environment": state["environment"],
        "version": state["version"],
        "message": "Approve production deployment?"
    })

    return {
        "approved": decision == "approve"
    }


def deploy(state: State):
    if not state["approved"]:
        return {
            "result": "Deployment rejected"
        }

    return {
        "result": (
            f"Deployed {state['version']} "
            f"to {state['environment']}"
        )
    }


graph = StateGraph(State)

graph.add_node("check_risk", check_risk)
graph.add_node("approval", request_approval)
graph.add_node("deploy", deploy)

graph.add_edge(START, "check_risk")
graph.add_edge("check_risk", "approval")
graph.add_edge("approval", "deploy")
graph.add_edge("deploy", END)

app = graph.compile()
```

Execution pauses at:

```python
interrupt(...)
```

The workflow can later resume with the human's decision.

Conceptually:

```text
LangGraph
    │
    ▼
Production Deploy Request
    │
    ▼
interrupt()
    │
    X
    │
    │ Human reviews
    │
    ▼
resume("approve")
    │
    ▼
Deploy
```

For production, combine this with a **durable checkpointer**, authentication, authorization, approval expiry, and audit logging.

---

## Approval vs Authorization

They solve different problems.

```text
Authorization
→ Is this actor allowed to perform this action?

Human Approval
→ Should this specific action be allowed right now?
```

Both can be required:

```text
Tool Request
    ↓
Authorization ✓
    ↓
High Risk
    ↓
Human Approval ✓
    ↓
Execute
```

A human should not be able to approve an action they are not authorized to approve.

---

## Approval vs Human-in-the-Loop

Human-in-the-loop is broader.

It can include:

* Approval
* Data correction
* Clarification
* Review
* Exception handling
* Manual intervention

Human approval is specifically a **decision gate** before an action.

---

## Production Considerations

A production approval system should handle:

* Authorized approvers
* Approval expiry
* Durable state
* Duplicate approvals
* Request cancellation
* Audit logging
* Authentication
* Authorization
* Sensitive-data redaction
* Approval context
* Timeout handling
* Re-validation before execution

Re-validate permissions after approval if significant time has passed.

---

## When to Use

Use human approval when:

* The action is irreversible
* Financial or legal impact exists
* Sensitive data is involved
* Production systems are affected
* Permissions are changed
* External communication has significant consequences

---

## When NOT to Use

Avoid approval gates for every action.

If every low-risk operation requires a human:

```text
Agent
 ↓
Human
 ↓
Agent
 ↓
Human
 ↓
Agent
```

the system loses most of its automation value.

Use **risk-based approval**, not universal approval.

---

## Trade-offs

| Benefit                       | Cost                 |
| ----------------------------- | -------------------- |
| Reduces high-impact mistakes  | Human latency        |
| Provides accountability       | Operational overhead |
| Controls irreversible actions | Approval bottlenecks |
| Supports regulated workflows  | More workflow state  |

---

## Mental Model

> **Let the agent automate low-risk work; require humans to control high-impact decisions.**

```text
Low Risk
   → Agent

High Risk
   → Agent proposes
   → Human decides
   → System executes
```
