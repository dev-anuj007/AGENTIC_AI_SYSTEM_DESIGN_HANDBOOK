# Plan and Execute Pattern

## 1. Problem

ReAct decides the next action incrementally:

```text id="pe01"
Reason → Act → Observe → Reason → Act
```

This works well for exploration, but some tasks are easier when the system first creates a **high-level plan** and then executes it.

Example:

> "Analyze this company and produce a security risk report."

The system may first determine:

```text id="pe02"
1. Discover assets
2. Identify vulnerabilities
3. Assess risk
4. Generate report
```

It can then execute those steps.

---

## 2. Core Idea

The pattern separates:

```text id="pe03"
Planning
   ↓
Execution
```

Architecture:

```text id="pe04"
                Goal
                  ↓
               Planner
                  ↓
                Plan
                  ↓
              Executor
                  ↓
          ┌───────┼───────┐
          ↓       ↓       ↓
         Step 1  Step 2  Step 3
          └───────┼───────┘
                  ↓
                Result
```

The planner determines **what needs to be done**.

The executor determines **how each planned step is performed**.

---

## 3. Example: Company Security Assessment

Goal:

> "Assess ACME's cybersecurity posture."

### Planning

The planner creates:

```text id="pe05"
Plan:
1. Discover public assets
2. Identify exposed services
3. Check known vulnerabilities
4. Analyze security configuration
5. Generate risk assessment
```

### Execution

```text id="pe06"
Plan
 ↓
Execute Step 1
 ↓
Execute Step 2
 ↓
Execute Step 3
 ↓
Execute Step 4
 ↓
Execute Step 5
 ↓
Report
```

The executor may use tools such as:

```text id="pe07"
DNS lookup
Port scanner
Vulnerability database
Web scanner
Report generator
```

---

## 4. Why Separate Planning and Execution?

The separation provides a useful architectural boundary.

### Planner

Responsible for:

* Decomposing the goal
* Determining required steps
* Establishing dependencies
* Selecting a strategy

### Executor

Responsible for:

* Running the steps
* Calling tools
* Managing step failures
* Producing outputs

This allows the system to inspect or modify the plan before expensive execution begins.

---

## 5. Plan Representation

A plan should ideally be structured rather than stored only as natural language.

Example:

```json id="pe08"
{
  "steps": [
    {
      "id": "discover_assets",
      "action": "asset_discovery"
    },
    {
      "id": "scan_services",
      "action": "service_scan",
      "depends_on": ["discover_assets"]
    },
    {
      "id": "check_vulnerabilities",
      "action": "vulnerability_scan",
      "depends_on": ["scan_services"]
    }
  ]
}
```

This makes the plan:

* Validatable
* Observable
* Executable
* Checkpointable

It also allows the runtime to determine which steps can execute concurrently.

---

## 6. Dynamic Planning

The initial plan does not always need to remain unchanged.

Suppose execution discovers:

```text id="pe09"
Exposed service = Apache
```

The system may add:

```text id="pe10"
Check Apache-specific vulnerabilities
```

Therefore, plan-and-execute systems can support:

```text id="pe11"
Plan
 ↓
Execute
 ↓
Observe
 ↓
Re-plan if necessary
 ↓
Execute
```

This creates a middle ground between a completely fixed workflow and a fully autonomous agent loop.

---

## 7. When to Use

Use Plan and Execute when:

* The task is complex enough to require decomposition.
* The overall objective is known.
* Multiple dependent steps are required.
* A high-level plan improves transparency or control.
* Execution can be separated from planning.

Typical examples:

```text id="pe12"
Research
Coding
Data analysis
Security assessment
Long-running business processes
Complex document processing
```

---

## 8. When Not to Use

Avoid this pattern when:

### The workflow is simple

```text id="pe13"
A → B → C
```

A normal sequential workflow is sufficient.

### The task is highly reactive

If every next action depends heavily on immediate observations, ReAct may be more appropriate.

### Planning provides little value

If creating a plan costs almost as much as executing the task, the additional planning step may not justify itself.

---

## 9. ReAct vs Plan and Execute

|              | ReAct                         | Plan and Execute         |
| ------------ | ----------------------------- | ------------------------ |
| Planning     | Incremental                   | Explicit                 |
| Execution    | Interleaved with reasoning    | Separate                 |
| Adaptability | High                          | Medium–High              |
| Visibility   | Lower                         | Higher                   |
| Long tasks   | Can become difficult to track | Easier to structure      |
| Best for     | Exploration                   | Complex multi-step goals |

Mental distinction:

```text id="pe14"
ReAct

Reason
 ↓
Act
 ↓
Observe
 ↓
Reason again


Plan & Execute

Plan
 ↓
Execute
 ↓
Re-plan if necessary
```

---

## 10. Production Considerations

### Plan Validation

Do not blindly execute an LLM-generated plan.

Validate:

```text id="pe15"
Are actions allowed?
Are dependencies valid?
Are required tools available?
Are parameters valid?
```

### Resource Limits

Plans should have limits for:

```text id="pe16"
max_steps
max_execution_time
max_tool_calls
max_tokens
max_cost
```

### Checkpointing

Long-running plans should persist progress:

```text id="pe17"
Step 1 ✓
Step 2 ✓
Step 3 ✗
Checkpoint
```

The system can resume from Step 3 instead of rebuilding the entire plan.

### Observability

Track:

```text id="pe18"
plan_id
step_id
status
dependencies
tool_calls
latency
errors
replans
cost
```

---

## 11. Trade-offs

| Aspect       | Plan and Execute                 |
| ------------ | -------------------------------- |
| Planning     | Explicit                         |
| Execution    | Structured                       |
| Adaptability | Medium–High                      |
| Transparency | High                             |
| Complexity   | Medium                           |
| Main benefit | Handles complex multi-step goals |
| Main risk    | Poor or unnecessary plans        |

The main trade-off is:

> **Explicit planning improves structure and visibility, but introduces planning cost and the possibility of an incorrect plan.**

---

## 12. Mental Model

```text id="pe19"
Goal
 ↓
Plan
 ↓
Validate
 ↓
Execute
 ↓
Observe
 ↓
 ┌───────────────┐
 │ Need re-plan? │
 └───────┬───────┘
     Yes │ No
         │
         ↓
      Re-plan
         │
         └──→ Execute

No → Complete
```

The key principle is:

> **Plan at the right level of abstraction, execute the plan deterministically where possible, and re-plan only when new information invalidates the current plan.**
