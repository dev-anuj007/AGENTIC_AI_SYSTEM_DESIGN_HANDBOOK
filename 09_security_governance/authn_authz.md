# Authentication and Authorization

## Problem

Agentic systems interact with:

* APIs
* Databases
* Internal services
* External tools
* Other agents
* Sensitive user data

The system must answer two different questions:

```text
Authentication
→ Who are you?

Authorization
→ What are you allowed to do?
```

An authenticated agent should **not automatically have access to every tool or resource**.

---

## Core Idea

```text id="k3m8qp"
User / Agent
     │
     ▼
Authentication
     │
     ▼
Identity
     │
     ▼
Authorization
     │
     ├── Allowed → Execute
     │
     └── Denied  → Reject
```

Authentication establishes identity.

Authorization evaluates whether that identity can perform a specific action on a specific resource.

---

## Example: Enterprise Coding Agent

Suppose an employee asks a coding agent:

> "Deploy the latest version to production."

The agent may have access to:

```text
Tools
├── read_repository
├── create_pull_request
├── run_tests
├── deploy_staging
└── deploy_production
```

Authentication identifies the user.

Authorization then evaluates:

```text
User
 │
 ├── Identity: user_123
 ├── Role: developer
 └── Tenant: company_a
        │
        ▼
Authorization
        │
        ├── read_repository ✓
        ├── run_tests ✓
        ├── deploy_staging ✓
        └── deploy_production ✗
```

The LLM may decide that production deployment is the next action, but the authorization layer must independently enforce whether that action is permitted.

---

## Agentic Authorization

Authorization should be evaluated at the **action boundary**, not only when the agent starts.

```text id="v6p2rx"
Agent
  │
  ▼
Tool Request
  │
  ▼
Authorization Check
  │
  ├── Allowed → Tool
  │
  └── Denied → Reject
```

This prevents an agent from gaining broader privileges simply because it is executing a long-running task.

---

## Authentication Methods

Depending on the architecture, identity can come from:

* OAuth/OIDC
* JWT
* API keys
* Service identities
* IAM roles
* mTLS
* Workload identity

The important property is that downstream services can reliably identify **who or what is making the request**.

---

## Authorization Models

Common models include:

### RBAC

Permissions are associated with roles.

```text
Developer
 → read_code
 → run_tests

Admin
 → read_code
 → run_tests
 → deploy_production
```

### ABAC

Permissions depend on attributes.

```text
Allow deployment if:

user.role == "developer"
AND
environment == "staging"
AND
repository.tenant == user.tenant
```

### Resource-Based Authorization

The resource itself defines who can access it.

```text
Repository A
 ├── Team X → read/write
 └── Team Y → read
```

Agentic systems often combine these approaches.

---

## Least Privilege

Give an agent only the permissions required for its task.

```text
Bad

Agent
 └── Full production access


Better

Agent
 ├── Read repository
 ├── Run tests
 └── Create PR
```

If production deployment is required, it can be placed behind a stronger authorization or human-approval boundary.

---

## Authentication vs Authorization

| Authentication        | Authorization                              |
| --------------------- | ------------------------------------------ |
| Establishes identity  | Determines permissions                     |
| "Who are you?"        | "What can you do?"                         |
| Happens before access | Evaluated when accessing resources/actions |
| Identity/token based  | Policy/permission based                    |

---

## Production Considerations

A production agentic system should consider:

* Identity propagation
* Least privilege
* Tenant isolation
* Tool-level permissions
* Resource-level permissions
* Short-lived credentials
* Token expiration
* Policy enforcement outside the LLM
* Audit logging
* Permission changes during execution

Never rely on the LLM to enforce authorization.

---

## When to Use

Authentication and authorization are required whenever an agent:

* Accesses user data
* Calls protected APIs
* Uses enterprise tools
* Executes privileged actions
* Accesses tenant-specific resources
* Communicates with other agents

---

## Mental Model

> **Authentication identifies the actor. Authorization controls what that actor is allowed to do.**

For agentic systems:

```text
LLM decides WHAT
      ↓
Authorization decides WHETHER
      ↓
Tool executes HOW
```
