# Agent as Tool Pattern

## Problem

A multi-agent system does not always need agents to communicate directly.

Sometimes one agent has a specialized capability that another agent needs. Instead of exposing all of that agent's internal reasoning and state, we can expose it as a **tool**.

The calling agent treats the specialized agent like any other callable capability.

---

## Core Idea

An agent can be wrapped behind a tool interface.

```text
                Main Agent
                    │
             tool: research()
                    │
                    ▼
             Research Agent
                    │
              Tools / RAG
                    │
                    ▼
              Research Result
                    │
                    ▼
                Main Agent
```

The main agent does not need to know how the research agent works internally.

It only needs to know:

```text
Input → Agent Tool → Structured Output
```

---

## Example: Software Engineering Agent

Suppose a coding agent needs to understand an unfamiliar codebase.

Instead of giving the coding agent direct access to every repository-analysis capability, expose a specialized **Codebase Analysis Agent** as a tool:

```text
Coding Agent
     │
     │ analyze_codebase()
     ▼
Codebase Analysis Agent
     │
     ├── Repository Search
     ├── Dependency Analysis
     └── Architecture Analysis
     │
     ▼
Structured Analysis
     │
     ▼
Coding Agent
```

The coding agent can then decide:

```text
"I need to understand the authentication flow"
        ↓
analyze_codebase("authentication flow")
        ↓
receive structured findings
        ↓
continue implementation
```

The internal reasoning and tools of the analysis agent remain encapsulated.

---

## Agent vs Agent-as-Tool

The important distinction is the **interaction model**.

| Direct Agent Collaboration          | Agent as Tool                                |
| ----------------------------------- | -------------------------------------------- |
| Agents communicate with each other  | Caller invokes an agent through an interface |
| More autonomous                     | More controlled                              |
| Shared/dynamic interaction possible | Request → response                           |
| More coordination complexity        | Simpler integration                          |
| Useful for collaboration            | Useful for specialized capabilities          |

An agent-as-tool is therefore closer to **capability composition** than open-ended multi-agent collaboration.

---

## Tool Contract

The agent should expose a clear interface:

```json
{
  "name": "analyze_codebase",
  "input": {
    "query": "authentication flow"
  },
  "output": {
    "summary": "...",
    "files": ["..."],
    "dependencies": ["..."]
  }
}
```

The caller should not depend on the internal implementation of the specialized agent.

This allows the underlying agent to evolve independently.

---

## When to Use

Use Agent-as-Tool when:

* One agent needs a specialized capability
* The capability has a clear input/output contract
* Internal agent complexity should be hidden
* You want reusable specialized agents
* The caller should remain responsible for the overall task

---

## When NOT to Use

Avoid it when:

* Agents need continuous two-way collaboration
* The sub-agent must independently coordinate multiple agents
* A central supervisor should control the entire execution
* The interaction requires shared evolving state

In those cases, **Supervisor** or **Peer-to-Peer** patterns may be more appropriate.

---

## Production Considerations

Treat the agent like a production tool:

* Define strict input/output schemas
* Enforce authentication and authorization
* Apply timeouts
* Limit token and cost budgets
* Handle retries carefully
* Trace every invocation
* Keep the agent's permissions scoped
* Validate returned data

The calling agent should not automatically inherit all permissions of the underlying agent.

---

## Trade-offs

**Benefits**

* Strong encapsulation
* Reusable specialized capabilities
* Simpler orchestration
* Easier permission boundaries

**Costs**

* Additional model invocation
* Extra latency and cost
* Context must cross an interface
* Less direct collaboration

---

## Mental Model

> **Agent as Tool = expose an agent's capability through a controlled interface.**

The key question is:

**Does the caller need another autonomous collaborator, or does it simply need a specialized capability?**

If it only needs the capability, **Agent as Tool** is often the simpler abstraction.
