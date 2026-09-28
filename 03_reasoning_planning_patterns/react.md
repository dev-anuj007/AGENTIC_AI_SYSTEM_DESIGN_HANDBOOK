# ReAct Pattern

## 1. Problem

Some tasks cannot be solved reliably with a single LLM call because the model needs to:

* Reason about the current state
* Take an action
* Observe the result
* Decide what to do next

For example, answering:

> "Why is the production API returning errors?"

may require searching logs, checking metrics, inspecting recent deployments, and then deciding what to investigate next.

A fixed workflow may not know the correct sequence beforehand.

---

## 2. Core Idea

**ReAct = Reason + Act**

The agent repeatedly alternates between reasoning and actions:

```text id="react01"
Goal
 ↓
Reason
 ↓
Act
 ↓
Observe
 ↓
Reason
 ↓
Act
 ↓
Observe
 ↓
...
 ↓
Finish
```

The important property is that the result of an action becomes context for the next decision.

---

## 3. Example: Production Incident Investigation

Goal:

> "Find the cause of the API latency spike."

The agent may execute:

```text id="react02"
Goal
 ↓
Reason: Need evidence
 ↓
Check metrics
 ↓
Observe: latency increased after deployment
 ↓
Reason: Inspect deployment
 ↓
Check recent deployment
 ↓
Observe: database query changed
 ↓
Reason: Verify database impact
 ↓
Check DB metrics
 ↓
Observe: query latency increased
 ↓
Conclude root cause
```

The exact sequence was not necessarily known beforehand.

The agent uses each observation to determine the next action.

---

## 4. Architecture

A ReAct system typically contains:

```text id="react03"
                    Goal
                     ↓
                  Agent
             ┌──────┴──────┐
             │   Reason    │
             └──────┬──────┘
                    ↓
               Select Tool
                    ↓
              Execute Tool
                    ↓
                 Result
                    │
                    └──────→ Agent
```

The agent maintains state containing information such as:

```text id="react04"
goal
conversation
previous_actions
tool_results
observations
current_progress
```

---

## 5. Why ReAct Is Different From a Fixed Workflow

A workflow might define:

```text id="react05"
Check Metrics
   ↓
Check Deployment
   ↓
Check Database
```

ReAct allows the next action to depend on what was discovered:

```text id="react06"
Check Metrics
     ↓
What did we learn?
     ↓
Choose next action
```

For example:

```text id="react07"
Metrics show CPU spike
       ↓
Investigate CPU

Metrics show DB latency
       ↓
Investigate Database
```

The execution path is therefore **adaptive**.

---

## 6. Tool Selection

The agent may have tools such as:

```text id="react08"
get_metrics()
get_logs()
get_deployments()
query_database()
search_documentation()
```

The model decides which tool is relevant based on the current state.

A simplified interaction:

```text id="react09"
Agent:
"I need deployment information."

       ↓

Tool:
get_deployments()

       ↓

Observation:
"Deployment v42 occurred 5 minutes ago."

       ↓

Agent:
"Now inspect the database changes."
```

---

## 7. Termination

A ReAct loop must have a clear stopping mechanism.

The agent can finish when:

```text id="react10"
Goal satisfied
```

or when a runtime limit is reached:

```text id="react11"
MAX_ITERATIONS
MAX_TOOL_CALLS
MAX_EXECUTION_TIME
MAX_TOKEN_BUDGET
MAX_COST
```

If the agent cannot reach a reliable conclusion, it should return an appropriate failure or escalation state rather than continue indefinitely.

---

## 8. When to Use

Use ReAct when:

* The next action depends on previous observations.
* Tool selection is dynamic.
* The task requires investigation or exploration.
* The execution path cannot be reliably predefined.
* Intermediate results influence subsequent decisions.

Typical examples:

```text id="react12"
Research agents
Incident investigation
Coding agents
Data analysis
Troubleshooting
Open-ended search
```

---

## 9. When Not to Use

Avoid ReAct when the process is already deterministic.

For example:

```text id="react13"
Validate
  ↓
Transform
  ↓
Store
```

A sequential workflow is simpler.

Also avoid unnecessary agentic reasoning when a direct tool call or simple conditional can solve the problem.

---

## 10. Production Considerations

### Tool Safety

The agent should only access tools permitted for its task.

```text id="react14"
Agent
 ↓
Permission Check
 ↓
Tool
```

### Tool Errors

A failed tool call becomes an observation that the agent can potentially recover from.

```text id="react15"
Tool
 ↓
Failure
 ↓
Agent
 ↓
Retry / Alternative Tool / Escalate
```

### Context Growth

Every observation can increase the agent's context.

Long-running loops therefore require strategies such as:

* Context compression
* Summarization
* State management
* Selective history retention

### Observability

Track:

```text id="react16"
iteration
reasoning step
tool selected
tool latency
tool result
errors
tokens
cost
termination reason
```

---

## 11. Trade-offs

| Aspect         | ReAct                              |
| -------------- | ---------------------------------- |
| Execution path | Dynamic                            |
| Tool selection | Dynamic                            |
| Adaptability   | High                               |
| Predictability | Lower                              |
| Debugging      | More difficult                     |
| Latency        | Variable                           |
| Cost           | Variable                           |
| Best for       | Investigation and open-ended tasks |
| Main risk      | Unnecessary or incorrect actions   |

The fundamental trade-off is:

> **More adaptability in exchange for less predictability and stronger runtime controls.**

---

## 12. ReAct vs Workflow

|                | Workflow           | ReAct               |
| -------------- | ------------------ | ------------------- |
| Next step      | Predefined         | Model-selected      |
| Execution      | Deterministic      | Adaptive            |
| Tool selection | Usually predefined | Dynamic             |
| Best for       | Known processes    | Open-ended tasks    |
| Control        | High               | Requires boundaries |

A useful architectural rule:

> **If you can confidently define the execution path beforehand, prefer a workflow. If the next action depends on runtime observations, ReAct becomes useful.**

---

## 13. Mental Model

```text id="react17"
Reason
  ↓
Act
  ↓
Observe
  ↓
Reason again
  ↓
...
```

The key idea is not simply "use an LLM with tools."

It is:

> **Use observations from the environment to dynamically determine the next action.**
