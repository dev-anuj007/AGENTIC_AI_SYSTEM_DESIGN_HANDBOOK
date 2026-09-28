# Hierarchical Multi-Agent Pattern

## Problem

A single supervisor can become difficult to manage when a system contains many agents or multiple levels of specialization.

A **hierarchical architecture** introduces multiple coordination levels.

---

## Core Idea

Instead of one supervisor coordinating every agent, supervisors delegate to other supervisors.

```text
                         User Goal
                            │
                            ▼
                    Global Supervisor
                     /              \
                    ▼                ▼
             Engineering Lead    Research Lead
              /          \          /       \
             ▼            ▼        ▼         ▼
          Coding         QA     Web Agent  Data Agent
```

Each coordinator manages a smaller part of the overall problem.

---

## Example: Build a Complete Product

Suppose the user asks:

> "Build a SaaS application for managing customer support."

The hierarchy could be:

```text
Global Supervisor
       │
       ├── Engineering Supervisor
       │      ├── Backend Agent
       │      ├── Frontend Agent
       │      └── QA Agent
       │
       └── Research Supervisor
              ├── Market Research Agent
              └── Competitor Research Agent
```

The **Global Supervisor** only needs to coordinate major workstreams.

The **Engineering Supervisor** handles engineering-specific decisions.

The **Research Supervisor** handles research-specific decisions.

This prevents the top-level supervisor from directly managing every individual agent.

---

## Responsibility Boundaries

Each level should have a clear scope.

```text
Global Supervisor
    → coordinates workstreams

Domain Supervisor
    → coordinates domain agents

Specialist Agent
    → performs the actual task
```

A supervisor should delegate rather than perform the work itself.

---

## When to Use

Use hierarchical orchestration when:

* There are many agents
* Agents naturally form domain groups
* Different domains require different coordination logic
* A single supervisor would become too complex
* Teams/workstreams can operate semi-independently

---

## When NOT to Use

Avoid it when:

* There are only a few agents
* The task has a simple execution path
* Agents do not have meaningful domain boundaries
* Coordination overhead exceeds the benefit

For a small multi-agent system, a single **Supervisor** is usually simpler.

---

## Trade-offs

| Benefit                        | Cost                                 |
| ------------------------------ | ------------------------------------ |
| Scales to many agents          | More coordination layers             |
| Clear domain boundaries        | More latency                         |
| Smaller context per supervisor | More complex state management        |
| Easier team/domain ownership   | Failures can propagate across levels |
| Better isolation               | Harder debugging                     |

The hierarchy should be introduced because it represents a real organizational or domain boundary—not simply because more agents exist.

---

## Production Considerations

Each supervisor should have:

* Clear scope and responsibilities
* Input/output contracts
* Permission boundaries
* Execution limits
* Failure-handling rules
* Checkpointed state
* Distributed tracing

Avoid passing the entire global context through every level. Pass only the information required for the delegated task.

---

## Mental Model

> **Use hierarchy when the number or complexity of agents becomes too large for a single coordinator.**

A useful progression is:

```text
Single Agent
     ↓
Supervisor + Agents
     ↓
Hierarchical Supervisors
```

The goal is not to maximize the number of agents. The goal is to keep **coordination manageable as system complexity grows**.
