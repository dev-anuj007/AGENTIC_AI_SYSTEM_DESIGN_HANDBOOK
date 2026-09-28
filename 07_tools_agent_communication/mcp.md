# Model Context Protocol (MCP)

## Problem

As agents use more external capabilities, integrating every tool directly into every agent becomes difficult.

Without a standard interface:

```text
Agent A → Custom integration → Database
Agent A → Custom integration → GitHub
Agent A → Custom integration → Slack

Agent B → Different integration → Database
Agent B → Different integration → GitHub
Agent B → Different integration → Slack
```

This creates duplicated integration logic.

**Model Context Protocol (MCP)** provides a standardized way for AI applications to connect to external tools and context providers.

---

## Core Idea

MCP separates the **AI application** from the systems providing capabilities.

```text id="w4m8kc"
                AI Application
                     │
                MCP Client
                     │
             ────────┼────────
                     │
                MCP Server
               /     |      \
              ▼      ▼       ▼
          Database  GitHub   APIs
```

The AI application communicates with an MCP server through a defined protocol instead of implementing every integration itself.

---

## Example: Software Engineering Agent

Suppose a coding agent needs access to:

* GitHub repositories
* Issue tracking
* Documentation
* Database information

Instead of building custom integrations into the agent, the platform can expose these capabilities through MCP servers:

```text id="q5v2nx"
Coding Agent
      │
      ▼
   MCP Client
      │
      ├── GitHub MCP Server
      │
      ├── Issue Tracker MCP Server
      │
      └── Documentation MCP Server
```

The agent can discover and invoke the capabilities exposed by those servers.

The coding agent remains focused on **reasoning about the software task**, while the integration layer handles access to external systems.

---

## MCP Concepts

At a high level, MCP can expose capabilities such as:

### Tools

Actions the model/application can invoke.

```text
create_issue()
search_repository()
run_query()
```

### Resources

Context or data that can be retrieved.

```text
repository files
documentation
database records
```

### Prompts

Reusable interaction templates exposed by an MCP server.

The important architectural idea is that these capabilities are exposed through a **standard protocol**.

---

## MCP vs Direct Tool Calling

| Direct Tool Calling                | MCP                                           |
| ---------------------------------- | --------------------------------------------- |
| Application defines integrations   | MCP standardizes the integration interface    |
| Often tightly coupled              | More decoupled                                |
| Each integration may be custom     | Common protocol                               |
| Simple for a small number of tools | Useful across many integrations               |
| Application owns integration logic | MCP servers can encapsulate integration logic |

MCP does not replace tool calling.

It provides a **standardized way to expose and consume tools/context**.

---

## Security Boundary

MCP does not automatically make a tool safe.

The system still needs:

```text id="x3v7pm"
Agent
  ↓
MCP Client
  ↓
Authorization
  ↓
MCP Server
  ↓
External System
```

Production systems should control:

* Authentication
* Authorization
* Tool permissions
* Data access
* Tenant isolation
* Rate limits
* Audit logging
* Input validation

A server exposing a destructive operation should not automatically make that operation available to every agent.

---

## When to Use

MCP is useful when:

* Multiple AI applications need the same integrations
* Many external tools need to be exposed
* Integration portability matters
* You want a standard interface between AI applications and capabilities
* Tool providers should own their integration logic

---

## When NOT to Use

A direct function/API integration may be simpler when:

* The application has only a few internal tools
* There is no need for interoperability
* The integration is tightly coupled to one application
* Introducing a protocol adds unnecessary complexity

---

## Production Considerations

Treat MCP servers as production services.

Consider:

* Authentication
* Authorization
* Tool allowlists
* Input validation
* Timeouts
* Rate limiting
* Audit logs
* Versioning
* Observability
* Failure handling

Also distinguish **discoverability** from **permission**:

> An agent knowing that a tool exists does not mean it is authorized to use it.

---

## Mental Model

> **MCP standardizes how AI applications connect to external tools and context.**

Think of it as an interoperability layer:

```text
AI Application
      ↓
  MCP Protocol
      ↓
Capability Provider
      ↓
External System
```

The main value is not "another way to call a function."

The value is **standardized, reusable connectivity between AI applications and external capabilities**.
