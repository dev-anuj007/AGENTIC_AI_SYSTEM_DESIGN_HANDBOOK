# Agent Loop

An agent is fundamentally an **iterative decision-making loop**.

Unlike a traditional request-response system:

```text
Request → Process → Response
```

an agent may need to perform multiple actions before it can complete the task:

```text
Goal
 ↓
Decide
 ↓
Act
 ↓
Observe
 ↓
Decide again
 ↓
Act
 ↓
...
 ↓
Complete
```

The agent loop is the core execution pattern behind most agentic systems.

---

# 1. Basic Agent Loop

The simplest agent loop is:

```text
                     ┌──────────────┐
                     │     Goal     │
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │    Decide    │
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │     Act      │
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   Observe    │
                     └──────┬───────┘
                            │
                            ▼
                     Goal achieved?
                       /         \
                     NO           YES
                     │             │
                     └──────┐      ▼
                            │    Result
                            │
                            └──→ Decide
```

The important part is the loop:

```text
Decide → Act → Observe → Decide
```

---

# 2. Components of the Loop

## Decide

The model receives the current context and determines what should happen next.

```text
Goal
 +
Instructions
 +
State
 +
Previous Results
       ↓
     Model
       ↓
   Next Action
```

Possible decisions:

```text
Call a tool
Ask the user
Retrieve information
Delegate to another agent
Continue reasoning
Finish the task
```

---

## Act

The selected action is executed.

For example:

```text
Agent
  ↓
Search Tool
  ↓
Search Engine
  ↓
Results
```

The action could be:

* Tool invocation
* Database query
* API call
* Code execution
* Another agent invocation
* Human approval request

---

## Observe

The agent receives the result of the action.

```text
Tool
  ↓
Result
  ↓
Agent State
  ↓
Model
```

The observation becomes part of the context for the next decision.

For example:

```text
Agent searched for:
"ACME recent vulnerabilities"

Result:
CVE-2025-XXXX found
```

The agent can now decide what to do next based on that result.

---

# 3. Example

Consider:

> "Find why ACME's API is failing."

The agent might execute:

```text
Goal
 │
 ▼
Decide
 │
 └── Check API health
        │
        ▼
      Result
      API returns 500
        │
        ▼
      Decide
        │
        └── Check logs
               │
               ▼
             Result
             Database timeout
               │
               ▼
             Decide
               │
               └── Check database
                      │
                      ▼
                    Result
                    Connection pool exhausted
                      │
                      ▼
                    Decide
                      │
                      └── Generate diagnosis
                             │
                             ▼
                           Done
```

Notice that the agent did not need a predefined sequence such as:

```text
Check API
→ Check logs
→ Check database
```

It determined the next action from the previous observation.

---

# 4. State in the Agent Loop

The agent needs state to maintain information between iterations.

A simplified state might look like:

```python
state = {
    "goal": "Find why ACME API is failing",
    "messages": [],
    "actions": [],
    "observations": [],
    "status": "running"
}
```

After each action:

```text
Before:

state
  ↓
Decide
  ↓
Action

After:

Action
  ↓
Observation
  ↓
Update State
  ↓
Decide again
```

Therefore:

```text
Agent Loop
    +
State
    =
Multi-step execution
```

---

# 5. Termination

An agent loop must have a clear termination condition.

Common termination conditions include:

### Goal achieved

```text
Goal
 ↓
Action
 ↓
Observation
 ↓
Goal achieved
 ↓
STOP
```

### No useful action remains

```text
Agent
 ↓
Evaluate
 ↓
Nothing useful to do
 ↓
STOP
```

### Maximum iterations

```text
iteration = 10
       ↓
MAX_ITERATIONS = 10
       ↓
STOP
```

### Timeout

```text
Agent execution
       ↓
Maximum execution time reached
       ↓
STOP
```

### Budget exceeded

```text
Token / Cost Budget
       ↓
Exceeded
       ↓
STOP
```

The runtime should enforce these limits rather than relying only on the model to stop itself.

---

# 6. Simple Implementation

A simplified agent loop can look like:

```python
def run_agent(goal):
    state = {
        "goal": goal,
        "history": []
    }

    for _ in range(MAX_ITERATIONS):

        decision = model.decide(state)

        if decision.type == "finish":
            return decision.result

        result = execute_tool(
            decision.tool,
            decision.arguments
        )

        state["history"].append({
            "decision": decision,
            "result": result
        })

    raise RuntimeError("Maximum iterations exceeded")
```

The important architecture is:

```text
Model
  ↓
Decision
  ↓
Runtime
  ↓
Tool
  ↓
Result
  ↓
State
  ↓
Model
```

The actual implementation can be much more sophisticated.

---

# 7. Agent Loop vs Workflow

A workflow usually has:

```text
A → B → C → D
```

An agent loop has:

```text
A → ?
     │
     ├── B
     ├── C
     └── D
          │
          ▼
          ?
```

The next action depends on the current state and observations.

This is the fundamental distinction:

```text
Workflow
    → Predefined transitions

Agent Loop
    → Dynamic transitions
```

---

# 8. Agent Loop with Multiple Tools

An agent may have many available tools:

```text
                 Agent
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
      Search       DB        API
        │          │          │
        └──────────┼──────────┘
                   ▼
                 Result
                   │
                   ▼
                 Agent
```

The model selects the appropriate tool based on the current state.

For example:

```text
Question
   ↓
Agent
   ↓
Need customer data?
   ↓
Database
   ↓
Need external information?
   ↓
Search
   ↓
Need to update system?
   ↓
API
```

The agent loop therefore enables **dynamic tool selection**.

---

# 9. Agent Loop with Multiple Steps

A single task may require many iterations:

```text
Goal
 │
 ▼
Decide
 │
 ▼
Tool A
 │
 ▼
Observe
 │
 ▼
Decide
 │
 ▼
Tool B
 │
 ▼
Observe
 │
 ▼
Decide
 │
 ▼
Tool C
 │
 ▼
Observe
 │
 ▼
Finish
```

The number of iterations does not need to be known beforehand.

This is one of the main differences between agentic execution and fixed workflows.

---

# 10. Agent Loop with Human Approval

The loop can pause for human intervention.

```text
                    Agent
                      │
                      ▼
                  Decide Action
                      │
                      ▼
                 High-risk?
                  /       \
                NO         YES
                │           │
                ▼           ▼
             Execute     Human
                           │
                     ┌─────┴─────┐
                     ▼           ▼
                  Approve       Reject
                     │           │
                     ▼           ▼
                  Execute       Stop
```

The human becomes another possible state transition in the agent loop.

---

# 11. Agent Loop with Another Agent

An agent can delegate part of the task to another agent.

```text
                 Main Agent
                     │
                     ▼
               Need research?
                     │
                     ▼
               Research Agent
                     │
                     ▼
                  Result
                     │
                     ▼
                 Main Agent
                     │
                     ▼
                 Continue
```

This becomes the foundation for **multi-agent architectures**.

---

# 12. Important Runtime Controls

A production agent loop should not be completely unrestricted.

Typical controls include:

```text
MAX_ITERATIONS
MAX_EXECUTION_TIME
MAX_TOOL_CALLS
MAX_TOKEN_BUDGET
MAX_COST
```

Example:

```python
if iteration >= MAX_ITERATIONS:
    stop()

if elapsed_time > MAX_EXECUTION_TIME:
    stop()

if tool_calls >= MAX_TOOL_CALLS:
    stop()

if estimated_cost >= MAX_COST:
    stop()
```

These controls belong to the **runtime**, not the model.

---

# 13. Failure Handling

An action can fail:

```text
Agent
  ↓
Tool
  ↓
Failure
```

The runtime can then decide:

```text
Failure
  │
  ├── Retry
  │
  ├── Fallback
  │
  ├── Ask another tool
  │
  ├── Escalate
  │
  └── Stop
```

This is the foundation for the reliability patterns covered later.

---

# 14. The Most Important Mental Model

The agent loop can be reduced to:

```text
              ┌─────────────────────┐
              │        Goal         │
              └──────────┬──────────┘
                         ▼
                  ┌──────────────┐
                  │    Decide    │
                  └──────┬───────┘
                         ▼
                  ┌──────────────┐
                  │     Act      │
                  └──────┬───────┘
                         ▼
                  ┌──────────────┐
                  │   Observe    │
                  └──────┬───────┘
                         ▼
                  ┌──────────────┐
                  │ Update State │
                  └──────┬───────┘
                         │
                         └────────→ Decide
```

The core principle is:

> **An agent repeatedly observes the current state, decides the next action, executes it, and uses the result to determine what to do next.**

---

# 15. Relationship to Later Patterns

The basic agent loop is the foundation for many advanced patterns:

```text
Agent Loop
    │
    ├── ReAct
    │
    ├── Plan-Execute
    │
    ├── Reflection
    │
    ├── Evaluator-Optimizer
    │
    ├── Agentic RAG
    │
    ├── Multi-Agent
    │
    └── Human-in-the-Loop
```

The difference between these patterns is primarily **how the decision-making and control logic inside the loop is structured**.

---

# Summary

```text
Agent Loop

Goal
 ↓
Decide
 ↓
Act
 ↓
Observe
 ↓
Update State
 ↓
Continue or Stop
```

| Component   | Responsibility                             |
| ----------- | ------------------------------------------ |
| **Goal**    | Defines what the agent needs to accomplish |
| **Decide**  | Determines the next action                 |
| **Act**     | Executes the selected action               |
| **Observe** | Receives the result                        |
| **State**   | Maintains information across iterations    |
| **Runtime** | Controls execution and termination         |

The most important distinction is:

```text
Workflow
    → Developer defines the transitions

Agent Loop
    → Model dynamically selects transitions
```

The agent is autonomous **within the boundaries enforced by the runtime, tools, policies, and guardrails**.
