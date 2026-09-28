# Tenant Isolation

## Problem

Multi-tenant agentic systems serve multiple organizations or customers from shared infrastructure.

A critical requirement is:

> **One tenant must never access another tenant's data, state, tools, or resources.**

A single missing filter can cause a cross-tenant data leak.

---

## Core Idea

Every request should carry a trusted tenant identity through the entire execution path.

```text id="t8r4km"
User
 │
 ▼
Authentication
 │
 ▼
Tenant Identity
 │
 ▼
Agent / Workflow
 │
 ├── Memory
 ├── RAG
 ├── Tools
 ├── Database
 └── Agent State
      │
      ▼
   Tenant Filter
      │
      ▼
Tenant-Scoped Data
```

Tenant isolation must exist at **every data and action boundary**, not only at the API gateway.

---

## Example: Enterprise Research Agent

Suppose the same platform serves:

```text id="p3x8va"
Company A
 ├── Documents
 ├── Projects
 └── Agent Memory

Company B
 ├── Documents
 ├── Projects
 └── Agent Memory
```

A user from Company A asks:

> "Search our security documents."

The retrieval request should carry:

```python id="n5j2qd"
filters = {
    "tenant_id": "company_a"
}
```

The vector database must enforce the filter:

```text id="w4c7mz"
Query
 │
 ▼
Vector DB
 │
 ├── Company A documents ✓
 ├── Company B documents ✗
 └── Global documents ✓
```

The LLM should never be responsible for deciding which tenant's documents are safe to retrieve.

---

## Tenant Context

Tenant identity should be established from a trusted source.

```text id="r9k3px"
JWT / Identity
      │
      ▼
tenant_id = company_a
      │
      ▼
Execution Context
      │
      ├── user_id
      ├── tenant_id
      ├── roles
      └── permissions
```

Do not trust a tenant ID supplied only inside the user's prompt or arbitrary tool arguments.

For example:

```text
❌ User says:
"Search tenant company_b."

✓ Authenticated context says:
tenant_id = company_a
```

The authorization layer should enforce the trusted identity.

---

## Isolation Boundaries

Tenant isolation may be implemented at different layers:

```text id="f2n8qc"
Application
    ↓
Agent State
    ↓
Memory
    ↓
RAG / Vector DB
    ↓
Database
    ↓
Tools / External APIs
```

Common strategies include:

* Tenant ID filtering
* Separate database schemas
* Separate databases
* Separate vector namespaces
* Separate storage buckets
* Tenant-scoped encryption keys

The appropriate level depends on the required isolation and risk.

---

## Agent Memory

Memory is especially important because it persists beyond a single request.

```text id="v5q1rs"
Agent Memory
     │
     ├── tenant_id
     ├── user_id
     ├── memory_type
     └── content
```

Memory retrieval should always be tenant-scoped.

Otherwise:

```text
Company A User
      │
      ▼
Memory Search
      │
      X
Company B Memory
```

can result in a cross-tenant leak.

---

## RAG Isolation

Every retrieval query should enforce authorization before returning context.

```text id="m8z3qa"
User Query
    │
    ▼
Tenant Context
    │
    ▼
Retriever
    │
    ▼
Tenant Filter
    │
    ▼
Authorized Documents
    │
    ▼
LLM
```

This should happen before documents reach the model.

---

## Tool Isolation

Tools can also expose tenant data.

For example:

```text id="q6p4wk"
CRM Tool
 ├── search_customers
 ├── update_customer
 └── delete_customer
```

The tool should receive trusted tenant context and enforce it independently.

The agent should not be able to simply change:

```json
{
  "tenant_id": "company_b"
}
```

to bypass authorization.

---

## Production Considerations

A production system should define:

* Trusted tenant identity
* Tenant-scoped authentication
* Tenant-aware authorization
* Database filters
* Vector DB filters/namespaces
* Memory isolation
* Object storage isolation
* Tool isolation
* Cache isolation
* Logging isolation
* Encryption boundaries
* Cross-tenant access testing

Be especially careful with caches.

```text
Cache Key

❌ user_query

✓ tenant_id:user_id:user_query
```

Otherwise one tenant's cached response could be returned to another tenant.

---

## When to Use

Tenant isolation is required when:

* Multiple organizations share the platform
* Users have organization-specific data
* Agents access private enterprise systems
* Memory persists across users
* RAG contains tenant-specific documents

---

## When NOT to Use

A tenant-isolation architecture may not be necessary for:

* Single-tenant systems
* Public data-only applications
* Completely isolated deployments where each customer has independent infrastructure

Even then, normal authorization is still required.

---

## Trade-offs

| Stronger Isolation       | Cost                                       |
| ------------------------ | ------------------------------------------ |
| Separate databases       | Higher infrastructure cost                 |
| Separate schemas         | More operational complexity                |
| Separate vector stores   | Higher resource usage                      |
| Logical tenant filtering | Lower cost but requires strict enforcement |

The key decision is the required **security boundary**, not simply infrastructure cost.

---

## Mental Model

> **Tenant identity must travel with the request, and every data/action boundary must enforce it.**

```text id="k2m7vx"
Trusted Tenant Identity
        ↓
Agent
        ↓
Memory / RAG / DB / Tools
        ↓
Tenant Authorization
        ↓
Allowed Data + Actions
```
