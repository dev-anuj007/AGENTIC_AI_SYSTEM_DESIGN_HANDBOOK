# Circuit Breaker Pattern

## Problem

Retries and fallbacks help recover from failures, but continuously calling a failing dependency can make the situation worse.

For example:

```text
Agent → Failing API
Agent → Retry
Agent → Retry
Agent → Retry
...
```

This wastes resources and can overload an already unhealthy service.

A **Circuit Breaker** temporarily stops calls to a failing dependency.

---

## Core Idea

A circuit breaker has three states:

```text
             ┌───────────┐
             │   CLOSED  │
             │ Normal    │
             └─────┬─────┘
                   │
             Failures exceed
               threshold
                   │
                   ▼
             ┌───────────┐
             │    OPEN   │
             │ Fail fast │
             └─────┬─────┘
                   │
             Wait / cooldown
                   │
                   ▼
             ┌───────────┐
             │ HALF-OPEN │
             │ Test call │
             └─────┬─────┘
                   │
             ┌─────┴─────┐
             ▼           ▼
          Success       Failure
             │             │
             ▼             ▼
          CLOSED          OPEN
```

---

## Example: Agent Calling a Vulnerability API

Suppose an agent repeatedly calls an external vulnerability service.

The service starts returning `503`.

Without a circuit breaker:

```text
Agent
 ↓
API → 503
 ↓
Retry
 ↓
API → 503
 ↓
Retry
 ↓
API → 503
```

With a circuit breaker:

```text
Initial failures
      ↓
Failure threshold reached
      ↓
Circuit OPEN
      ↓
Requests fail immediately
      ↓
Cooldown
      ↓
HALF-OPEN
      ↓
Test request
      │
      ├── Success → CLOSED
      └── Failure → OPEN
```

The agent can use a fallback while the circuit is open.

---

## Circuit States

### Closed

Requests flow normally.

Failures are tracked.

### Open

Requests are rejected immediately without calling the dependency.

This protects both the agent system and the failing dependency.

### Half-Open

After a cooldown period, a limited number of requests are allowed to test whether the dependency has recovered.

---

## LangGraph Implementation

The circuit breaker can be represented as state plus conditional routing.

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    circuit: str
    failure_count: int
    threshold: int
    api_result: str


def check_circuit(state: State):
    # Circuit is already open
    if state["circuit"] == "open":
        return state

    return state


def call_api(state: State):
    try:
        # Replace with real vulnerability API call
        result = call_vulnerability_api()

        return {
            "api_result": result,
            "circuit": "closed",
            "failure_count": 0,
        }

    except Exception:
        failures = state["failure_count"] + 1

        if failures >= state["threshold"]:
            return {
                "circuit": "open",
                "failure_count": failures,
                "api_result": "API unavailable",
            }

        return {
            "failure_count": failures,
            "api_result": "temporary failure",
        }


def fallback(state: State):
    return {
        "api_result": "Using fallback vulnerability source"
    }


def route_after_check(state: State):
    if state["circuit"] == "open":
        return "fallback"

    return "call_api"


graph = StateGraph(State)

graph.add_node("check_circuit", check_circuit)
graph.add_node("call_api", call_api)
graph.add_node("fallback", fallback)

graph.add_edge(START, "check_circuit")

graph.add_conditional_edges(
    "check_circuit",
    route_after_check,
    {
        "call_api": "call_api",
        "fallback": "fallback",
    },
)

graph.add_edge("call_api", END)
graph.add_edge("fallback", END)

app = graph.compile()
```

The important part is not the syntax. The architecture is:

```text
              START
                │
                ▼
         Check Circuit
            │       │
         CLOSED     OPEN
            │         │
            ▼         ▼
        Call API    Fallback
            │         │
            └────┬────┘
                 ▼
                END
```

The circuit state lives in the graph state, while the conditional edge determines whether the dependency can be called.

---

## Retry + Circuit Breaker

In a real agentic system, retries can happen **inside the API node**, while the circuit breaker controls whether the node should be entered at all.

```text
                 Check Circuit
                       │
              ┌────────┴────────┐
              │                 │
            OPEN              CLOSED
              │                 │
              ▼                 ▼
          Fallback          Call API
                                │
                           ┌────┴────┐
                           │         │
                        Success    Failure
                           │         │
                           ▼         ▼
                          END    Retry / Count
                                      │
                              threshold reached?
                                      │
                                      ▼
                                   OPEN
```

This gives each mechanism a clear responsibility:

```text
Retry
→ Recover from an individual transient failure

Circuit Breaker
→ Stop calling an unhealthy dependency

Fallback
→ Continue using an alternative capability
```

---

## When to Use

Use circuit breakers when:

* A dependency is frequently unavailable
* Agents generate high request volume
* A failing dependency can consume significant resources
* Cascading failures are possible
* The system has a meaningful fallback

---

## When NOT to Use

A circuit breaker may be unnecessary when:

* The dependency is rarely called
* Failures have little system-wide impact
* There is no meaningful recovery/fallback behavior
* The operation is already asynchronous and isolated

---

## Production Considerations

A production LangGraph implementation should additionally handle:

* Persistent circuit state
* Failure windows rather than only a simple counter
* Cooldown timestamps
* Half-open probing
* Concurrent executions
* Timeouts
* Retry limits
* Fallbacks
* Metrics and alerts

For a distributed deployment, the circuit state should generally live in a shared state store rather than only inside one process.

For example:

```text
              LangGraph Workers
             /       |        \
            ▼        ▼         ▼
         Worker   Worker    Worker
             \       |        /
              ▼      ▼       ▼
              Shared State
                  Store
                    │
                    ▼
             Circuit State
```

Otherwise, one worker may believe the circuit is `OPEN` while another still believes it is `CLOSED`.

---

## Trade-offs

| Benefit                                        | Cost                                  |
| ---------------------------------------------- | ------------------------------------- |
| Prevents cascading failures                    | Requires state management             |
| Reduces unnecessary traffic                    | Requires threshold tuning             |
| Protects unhealthy dependencies                | May temporarily reject valid requests |
| Works naturally with conditional graph routing | Distributed state adds complexity     |

---

## Mental Model

> **Circuit Breaker = stop repeatedly calling a dependency that is currently unhealthy.**

In LangGraph:

```text
State
  ↓
Conditional Edge
  ↓
Allow / Block Dependency
  ↓
Retry / Fallback
```

The key principle is:

**Retry recovers from failure; the circuit breaker prevents repeated failure; fallback provides an alternative path.**
