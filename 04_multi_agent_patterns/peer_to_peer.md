# Peer-to-Peer Multi-Agent Pattern

## Problem

In a supervisor architecture, agents communicate through a central coordinator.

But sometimes agents need to communicate **directly with each other**.

For example, one agent may discover information that another agent needs, without requiring a supervisor to relay every message.

This is the **Peer-to-Peer Pattern**.

---

## Core Idea

Agents are peers that can communicate and delegate work directly.

```text id="3j4m8q"
              Agent A
             ↙       ↘
         Agent B ←→ Agent C
             ↘       ↙
              Agent D
```

There is no single agent responsible for coordinating every interaction.

Each agent can decide when another agent's capability is required.

---

## Example: Incident Investigation

Suppose an AI system is investigating a production outage.

```text id="0kqz3d"
Monitoring Agent
      │
      ├── discovers high API latency
      │
      ▼
Service Agent
      │
      ├── identifies database errors
      │
      ▼
Database Agent
      │
      ├── identifies connection exhaustion
      │
      ▼
Infrastructure Agent
      │
      └── recommends scaling database connections
```

The agents exchange findings directly as the investigation progresses.

The next agent is selected based on the current discovery rather than a predefined central workflow.

---

## Communication Model

Peer agents need explicit communication contracts.

```text id="4v9m2r"
Agent A
   │
   │ request
   ▼
Agent B
   │
   │ response
   ▼
Agent A
```

Messages should contain structured information such as:

```json id="7h3w9p"
{
  "task": "analyze_database_errors",
  "context": "...",
  "expected_output": "root_cause"
}
```

Avoid unrestricted agent-to-agent conversations because they can quickly become difficult to control and observe.

---

## When to Use

Use peer-to-peer coordination when:

* Agents are relatively autonomous
* Collaboration is dynamic
* Agents may discover new dependencies during execution
* No single coordinator naturally owns the entire process
* Agents need to react directly to each other's results

---

## When NOT to Use

Avoid it when:

* A central coordinator provides clearer control
* The workflow is predictable
* Strong governance and authorization are required
* The number of agents is large enough to create communication complexity

For highly controlled enterprise workflows, a **Supervisor** or **Hierarchical** architecture may be easier to govern.

---

## Trade-offs

| Benefit                              | Cost                             |
| ------------------------------------ | -------------------------------- |
| Highly flexible collaboration        | Harder to reason about           |
| No central bottleneck                | Communication can grow rapidly   |
| Agents can react directly            | More difficult debugging         |
| Naturally supports dynamic discovery | Harder authorization             |
| Decentralized coordination           | Risk of cycles and repeated work |

With `N` agents, unrestricted communication can create many possible interaction paths.

Therefore, production systems usually restrict **who can communicate with whom**.

---

## Production Considerations

Define:

* Agent identity
* Allowed communication partners
* Message schemas
* Timeouts
* Retry limits
* Correlation IDs
* Maximum interaction depth
* Authorization policies
* Full message tracing

Also detect cycles:

```text
Agent A → Agent B → Agent C → Agent A
```

Without limits, this can create infinite execution or unnecessary cost.

---

## Mental Model

> **Supervisor:** agents communicate through a coordinator.

> **Peer-to-Peer:** agents communicate directly when collaboration requires it.

The key design question is:

**Does coordination naturally belong to one central component, or should responsibility be distributed among the agents?**
