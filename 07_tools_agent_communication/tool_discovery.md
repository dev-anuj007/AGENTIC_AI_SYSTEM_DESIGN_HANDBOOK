# Tool Discovery

## Problem

An agent may have access to hundreds or thousands of tools.

Providing every tool definition to the model on every request creates:

* Large context
* Higher token cost
* Slower inference
* More tool-selection confusion

The system therefore needs a way to discover **only the capabilities relevant to the current task**.

---

## Core Idea

Instead of exposing every tool upfront:

```text id="q8n4mp"
User Request
     │
     ▼
Tool Discovery
     │
     ▼
Relevant Tools
     │
     ▼
Agent
     │
     ▼
Tool Execution
```

Tool discovery becomes a separate capability from tool execution.

---

## Example: Enterprise Engineering Agent

Suppose an engineering platform has:

```text id="f5q2nx"
500+ Tools
├── GitHub
├── Jira
├── AWS
├── Kubernetes
├── Databases
├── Monitoring
└── Internal APIs
```

A user asks:

> "Why is payment-api returning 500 errors?"

The system does not expose all 500 tools.

Discovery identifies relevant capabilities:

```text id="m7v3pk"
User Request
     │
     ▼
Tool Discovery
     │
     ├── query_logs()
     ├── get_service_health()
     └── query_metrics()
             │
             ▼
          Agent
             │
             ▼
        Investigate
```

If the agent later discovers that a Kubernetes deployment is involved, it can discover additional Kubernetes tools.

---

## Discovery vs Execution

These are separate responsibilities:

```text id="d6x9qw"
Discovery
→ "What capabilities are available?"

Execution
→ "Execute this specific capability."
```

A discovered tool is **not automatically authorized**.

```text id="p3r7vz"
Discover Tool
      ↓
Check Permission
      ↓
Execute Tool
```

---

## Discovery Strategies

### Static Registry

Maintain a catalog containing:

```text id="x5k2mr"
tool_name
description
capabilities
input_schema
permissions
owner
```

### Semantic Discovery

Use embeddings or search to find tools relevant to the user's task.

```text id="z7q4nc"
Query
 ↓
Tool Search
 ↓
Relevant Tool Definitions
```

### Capability-Based Discovery

Filter tools using structured capabilities:

```text id="v8m2kp"
domain = "payments"
operation = "read"
environment = "production"
```

Production systems can combine these approaches.

---

## When to Use

Use tool discovery when:

* The system has many tools
* Tools are dynamically added
* Multiple domains exist
* Context size matters
* Different agents require different capabilities

---

## When NOT to Use

Avoid dynamic discovery when:

* There are only a few tools
* The available tools are stable
* Tool selection is deterministic
* Discovery adds more complexity than it removes

---

## Production Considerations

Tool discovery should include:

* Permission filtering
* Tenant isolation
* Tool versioning
* Capability metadata
* Tool health/status
* Usage tracking
* Audit logging
* Schema validation

Most importantly:

> **Never discover first and authorize later as an afterthought.**

The discovery layer should already hide tools the agent is not permitted to use.

---

## Trade-offs

| Benefit                        | Cost                      |
| ------------------------------ | ------------------------- |
| Smaller model context          | Additional discovery step |
| Supports large tool ecosystems | Discovery latency         |
| Easier tool management         | More infrastructure       |
| Dynamic capabilities           | More complex evaluation   |

---

## Mental Model

```text id="c1v8mx"
Tool Registry
      ↓
Discover Relevant Capabilities
      ↓
Authorize
      ↓
Agent Selects Tool
      ↓
Execute
```

> **Tool discovery answers: "What can I use for this task?"**

> **Tool calling answers: "Use this capability now."**
