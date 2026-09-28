# Tool Permissions

## Problem

An agent may have access to many tools, but not every tool should be available for every task.

For example:

```text
Agent
├── search_documents
├── read_customer
├── update_customer
├── send_email
├── deploy_production
└── delete_database
```

Giving the agent unrestricted access creates unnecessary risk.

---

## Core Idea

Tool permissions define:

> **Which agent can use which tool, with which parameters, on which resources, under which conditions.**

```text
Agent
  │
  ▼
Tool Request
  │
  ▼
Permission Check
  │
  ├── Allowed ──→ Tool
  │
  └── Denied ───→ Reject
```

The LLM should never be the final authority for tool permissions.

---

## Example: Coding Agent

A coding agent may perform normal development tasks:

```text
Tools
├── read_repository       ✓
├── run_tests             ✓
├── create_pull_request   ✓
├── deploy_staging        ✓
└── deploy_production     ✗
```

The agent can request:

```json
{
  "tool": "deploy_production",
  "arguments": {
    "version": "v2.4.1"
  }
}
```

But the tool gateway evaluates the request before execution.

```text
Agent
  │
  ▼
deploy_production
  │
  ▼
Permission Check
  │
  ├── Developer role ✗
  │
  └── Request denied
```

---

## Permission Dimensions

Permissions can be defined across several dimensions:

```text
Agent
 │
 ├── Tool
 ├── Action
 ├── Resource
 ├── Tenant
 ├── Environment
 └── Time / Context
```

For example:

```text
Allow:

agent = coding_agent
tool = deploy
environment = staging
repository = tenant-owned
action = deploy
```

This is more precise than simply saying:

```text
coding_agent → deploy = allowed
```

---

## Read vs Write Permissions

Separate read and write capabilities.

```text
Database Agent

read_customer       ✓
search_customer     ✓
update_customer     ✗
delete_customer     ✗
```

Read operations generally have lower impact than destructive or externally visible operations.

This enables least-privilege design.

---

## Practical Permission Flow

```text
Agent
  │
  ▼
Tool Request
  │
  ▼
Authentication
  │
  ▼
Authorization
  │
  ▼
Policy Evaluation
  │
  ├── Allowed ──→ Validate Arguments
  │                    │
  │                    ▼
  │                  Execute
  │
  └── Denied ───→ Audit + Reject
```

Authorization answers **who can perform the action**.

Policy evaluation can additionally consider **context and risk**.

---

## Dynamic Permissions

Some permissions may depend on runtime conditions.

Example:

```text
Developer
   │
   ├── Deploy staging → Allowed
   │
   └── Deploy production
          │
          ▼
     Approval required
```

Another example:

```text
Business Hours
   │
   ├── Refund < ₹10,000 → Automatic
   │
   └── Refund ≥ ₹10,000 → Human approval
```

The permission system should enforce these rules independently of the model.

---

## Concrete Policy Example

A policy engine can represent permissions as structured rules.

For example:

```json
{
  "agent": "coding_agent",
  "rules": [
    {
      "tool": "read_repository",
      "action": "read",
      "environment": "*",
      "decision": "allow"
    },
    {
      "tool": "deploy",
      "action": "deploy",
      "environment": "staging",
      "decision": "allow"
    },
    {
      "tool": "deploy",
      "action": "deploy",
      "environment": "production",
      "requires_approval": true
    }
  ]
}
```

When the agent requests:

```json
{
  "tool": "deploy",
  "arguments": {
    "environment": "production",
    "version": "v2.4.1"
  }
}
```

the policy engine evaluates the request:

```text
Agent Request
     │
     ▼
Policy Engine
     │
     ├── Agent allowed? ✓
     ├── Tool allowed? ✓
     ├── Environment allowed? ✓
     └── Approval required? ✓
              │
              ▼
        Human Approval
              │
        ┌─────┴─────┐
        ▼           ▼
     Approved     Rejected
        │             │
        ▼             ▼
     Execute        Deny
```

A simplified implementation could look like:

```python
def authorize(request, policy):
    rule = find_matching_rule(request, policy)

    if not rule:
        return {
            "allowed": False,
            "reason": "No matching permission"
        }

    if rule.get("requires_approval"):
        return {
            "allowed": False,
            "requires_approval": True
        }

    return {
        "allowed": rule["decision"] == "allow"
    }
```

The important architecture is:

```text
LLM
 ↓
Tool Request
 ↓
Policy Engine
 ↓
Approval / Authorization
 ↓
Tool Execution
```

The LLM proposes the action; the policy engine makes the security decision.

---

## End-to-End LangGraph Implementation

A practical implementation places the policy check **between the agent's decision and the actual tool execution**.

```text
Agent
  │
  ▼
Tool Request
  │
  ▼
Policy Check
  │
  ├── Denied ─────────→ End
  │
  ├── Approval Needed → Human Approval
  │                         │
  │                    ┌────┴────┐
  │                    ▼         ▼
  │                 Approved   Rejected
  │                    │          │
  │                    ▼          ▼
  └──────────────→ Execute      End
                       │
                       ▼
                     Result
```

### State

```python
from typing import TypedDict


class State(TypedDict):
    user_role: str
    tool: str
    arguments: dict
    permission: str
    approval_required: bool
    approved: bool
    result: str
```

### Policy

```python
POLICY = {
    "developer": {
        "read_repository": {
            "allowed": True
        },
        "deploy": {
            "staging": {
                "allowed": True
            },
            "production": {
                "allowed": True,
                "requires_approval": True
            }
        }
    }
}
```

### Policy Check

```python
def check_permission(state: State):
    role = state["user_role"]
    tool = state["tool"]
    environment = state["arguments"].get("environment")

    rule = POLICY.get(role, {}).get(tool)

    if not rule:
        return {
            "permission": "denied",
            "approval_required": False
        }

    if environment:
        rule = rule.get(environment)

        if not rule:
            return {
                "permission": "denied",
                "approval_required": False
            }

    if rule.get("requires_approval"):
        return {
            "permission": "allowed",
            "approval_required": True
        }

    if rule.get("allowed"):
        return {
            "permission": "allowed",
            "approval_required": False
        }

    return {
        "permission": "denied",
        "approval_required": False
    }
```

### Human Approval

```python
def human_approval(state: State):
    # In production this would pause the workflow
    # and wait for an external approval event.

    approved = get_human_approval()

    return {
        "approved": approved
    }
```

### Tool Execution

```python
def execute_tool(state: State):
    if state["tool"] == "deploy":
        return {
            "result": deploy(
                environment=state["arguments"]["environment"],
                version=state["arguments"]["version"]
            )
        }

    return {
        "result": "Tool executed"
    }
```

### Routing

```python
def route_after_permission(state: State):
    if state["permission"] == "denied":
        return "end"

    if state["approval_required"]:
        return "approval"

    return "execute"


def route_after_approval(state: State):
    if state["approved"]:
        return "execute"

    return "end"
```

### Build the LangGraph

```python
from langgraph.graph import StateGraph, START, END


graph = StateGraph(State)

graph.add_node("check_permission", check_permission)
graph.add_node("human_approval", human_approval)
graph.add_node("execute_tool", execute_tool)

graph.add_edge(START, "check_permission")

graph.add_conditional_edges(
    "check_permission",
    route_after_permission,
    {
        "approval": "human_approval",
        "execute": "execute_tool",
        "end": END,
    }
)

graph.add_conditional_edges(
    "human_approval",
    route_after_approval,
    {
        "execute": "execute_tool",
        "end": END,
    }
)

graph.add_edge("execute_tool", END)

app = graph.compile()
```

A production request might look like:

```python
state = {
    "user_role": "developer",
    "tool": "deploy",
    "arguments": {
        "environment": "production",
        "version": "v2.4.1"
    },
    "permission": "",
    "approval_required": False,
    "approved": False,
    "result": ""
}

result = app.invoke(state)
```

The resulting execution is:

```text
Developer
    │
    ▼
deploy(production)
    │
    ▼
Policy Check
    │
    ├── Allowed ✓
    └── Approval Required ✓
             │
             ▼
       Human Approval
             │
             ▼
          Approved
             │
             ▼
       Execute Deploy
             │
             ▼
           Result
```

The critical security property is that **the agent cannot skip the policy node and directly invoke the deployment function**.

In production, the policy engine would normally be a dedicated service or authorization library, and human approval would pause the durable workflow until an external approval event resumes it.

---

## Tool Permissions vs Prompt Injection

Prompt injection can manipulate the model into requesting a dangerous tool.

Tool permissions provide the second security boundary:

```text
Malicious Document
       ↓
Agent Manipulated
       ↓
Dangerous Tool Request
       ↓
Permission Check
       ↓
Denied
```

Therefore:

> **Prompt security protects reasoning; tool permissions protect execution.**

---

## Least Privilege

Start with no permissions and explicitly grant required capabilities.

```text
❌ Agent → all tools

✓ Agent → required tools only
```

For example:

```text
Research Agent
├── search_web
├── search_documents
└── read_database

No:
├── delete_database
├── deploy_production
└── send_external_email
```

---

## Production Considerations

A production permission system should support:

* Authentication
* RBAC / ABAC
* Tool-level permissions
* Resource-level permissions
* Tenant isolation
* Environment restrictions
* Parameter validation
* Least privilege
* Short-lived credentials
* Human approval
* Audit logging
* Permission versioning
* Deny-by-default behavior

Permissions should be enforced server-side, not through the system prompt.

---

## When to Use

Tool permissions are essential when agents:

* Access enterprise systems
* Modify data
* Execute code
* Send external communications
* Access secrets
* Perform financial operations
* Deploy infrastructure
* Work across multiple tenants

---

## When NOT to Use

Complex permission controls may be unnecessary when an agent:

* Has no tools
* Only generates text
* Processes non-sensitive public information
* Runs in a completely isolated environment

Basic authorization may still be required.

---

## Trade-offs

| Benefit                             | Cost                             |
| ----------------------------------- | -------------------------------- |
| Limits agent blast radius           | Permission management            |
| Supports least privilege            | Policy complexity                |
| Protects high-risk tools            | Additional authorization latency |
| Enables tenant/environment controls | More testing                     |

---

## Mental Model

```text
LLM
 ↓
What should I do?
 ↓
Tool Request
 ↓
Permission System
 ↓
Am I allowed to do it?
 ↓
Tool
```

> **The agent chooses what it wants to do; the permission system decides what it is allowed to do.**
