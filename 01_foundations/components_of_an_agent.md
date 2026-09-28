# Components of an Agent

An agentic system is more than an LLM. The LLM provides reasoning capabilities, while the surrounding components provide **context, actions, execution, and control**.

A useful mental model is:

```text
                         Agent
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
      Model              State              Tools
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                           ▼
                     Orchestration
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                Guardrails     Memory
```

Additional production concerns such as **observability, evaluation, security, and reliability** sit around the agent rather than being part of its core reasoning loop.

---

## 1. Model

The **model** is the reasoning engine of the agent.

It determines:

* What should happen next?
* Which tool should be used?
* Whether more information is required?
* Whether the task is complete?

```text
User Request
     │
     ▼
   Model
     │
     ▼
Decision
```

The model decides what should happen, but normally does not execute external actions itself.

```text
Model = Reasoning
```

---

## 2. Instructions

Instructions define **how the agent should behave**.

They typically contain:

* Role
* Objective
* Constraints
* Behavioral rules
* Tool usage policies
* Output requirements

Example:

```text
You are a research agent.

Your responsibility is to research the requested topic
and provide an evidence-based answer.

Rules:
- Use available search tools when information may be outdated.
- Cite sources.
- Do not invent information.
```

The model uses instructions together with the user's request and current state.

```text
Instructions
      +
User Request
      +
State
      ↓
    Model
```

---

## 3. State

State represents the information required during the current execution.

For example:

```python
state = {
    "request": "...",
    "messages": [],
    "tool_results": [],
    "current_step": "research",
    "status": "running"
}
```

State allows an agent to maintain continuity across multiple steps.

```text
Step 1
  ↓
State
  ↓
Step 2
  ↓
State
  ↓
Step 3
```

### State vs Memory

State generally represents the **current execution**.

Memory represents information that may persist beyond the current execution.

This distinction becomes important in the `06_memory_state` chapter.

---

## 4. Tools

Tools give an agent the ability to interact with external systems.

Examples:

```text
Search
Database
API
File System
Code Execution
Internal Services
```

```text
                  Agent
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
        Search      DB        API
```

The model decides **which action is required**, while the runtime executes the tool.

A tool normally has:

```text
Name
Description
Input Schema
Execution Logic
Output
```

Tool architecture is covered in detail in:

```text
07_tools_agent_communication/
```

---

## 5. Execution Runtime

The execution runtime is responsible for **executing the decisions made by the model**.

```text
Model
  │
  │ "Call search_customer"
  ▼
Runtime
  │
  ▼
Tool
  │
  ▼
Result
  │
  ▼
Runtime
  │
  ▼
Model
```

The runtime may handle:

* Tool execution
* State updates
* Execution loops
* Timeouts
* Retries
* Checkpoints
* Termination

This is the layer where an agent becomes an **executable system rather than just an LLM prompt**.

---

## 6. Orchestration

Orchestration determines **how different execution steps are connected**.

A simple agent might have:

```text
Request
   ↓
Model
   ↓
Tool
   ↓
Model
   ↓
Response
```

A complex system might have:

```text
                    Router
                   /      \
                  ▼        ▼
             Research     Coding
                Agent      Agent
                  \        /
                   ▼      ▼
                    Evaluator
                        │
                        ▼
                     Response
```

Orchestration can be:

* Sequential
* Conditional
* Parallel
* Dynamic
* Multi-agent

These patterns are covered in the workflow and multi-agent chapters.

---

## 7. Memory

Memory allows an agent to retain information beyond its immediate context.

Examples:

```text
User preferences
Previous interactions
Historical information
Past tasks
Learned information
```

A simplified model:

```text
                 Agent
                   │
          ┌────────┴────────┐
          ▼                 ▼
        State             Memory
     Current task      Persistent data
```

Memory can use databases, vector stores, caches, or other persistence mechanisms.

Detailed memory architectures are covered in:

```text
06_memory_state/
```

---

## 8. Guardrails

Guardrails constrain and validate agent behavior.

They can operate at different points:

```text
User Input
    │
    ▼
Input Guardrail
    │
    ▼
   Agent
    │
    ▼
Tool Permission
    │
    ▼
Tool Execution
    │
    ▼
Output Guardrail
    │
    ▼
Response
```

Examples:

* Input validation
* Tool permissions
* Output validation
* Policy enforcement
* Human approval

Guardrails become especially important when agents can perform real-world actions.

---

## 9. Human-in-the-Loop

Some actions require human approval.

For example:

```text
Agent
  │
  ▼
Create Refund
  │
  ▼
Amount > $5,000
  │
  ▼
Human Approval
  │
  ├── Approve → Execute
  │
  └── Reject  → Stop
```

Human intervention can be triggered by:

* High-risk actions
* High-value operations
* Low confidence
* Policy violations
* Ambiguous requests

Human-in-the-loop is therefore a **control mechanism for agent autonomy**.

---

## 10. Observability

Agent execution can involve many dynamic operations:

```text
Request
  │
  ▼
Agent
  ├── Model Call
  ├── Tool Call
  ├── Retrieval
  ├── Tool Call
  ├── Retry
  └── Final Response
```

Observability allows us to understand what happened.

Important signals include:

* Agent execution trace
* Model calls
* Tool calls
* Latency
* Token usage
* Errors
* Retries
* Final outcome

Detailed observability patterns are covered in:

```text
10_observability/
```

---

# Putting Everything Together

A production agent can be viewed as:

```text
                         User
                           │
                           ▼
                    ┌─────────────┐
                    │ Instructions│
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │    Model    │
                    └──────┬──────┘
                           │
                    Decision / Action
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
            Tools        Memory       Human
              │            │          Approval
              └────────────┼────────────┘
                           ▼
                         State
                           │
                           ▼
                       Runtime
                           │
                           ▼
                     Continue / Stop
                           │
                           ▼
                        Response
```

Around this core execution we add:

```text
Security
Reliability
Observability
Evaluation
```

---

# The Core Mental Model

When designing an agent, think about:

```text
THINK
  → Model

CONTEXT
  → Instructions + State + Memory

ACT
  → Tools

EXECUTE
  → Runtime + Orchestration

CONTROL
  → Guardrails + Human Approval

OBSERVE
  → Observability
```

This gives us a useful distinction:

```text
Agent
  = Model + Context + Tools + Execution

Production Agentic System
  = Agent
    + Memory
    + Guardrails
    + Human Control
    + Observability
    + Reliability
    + Evaluation
```

The exact implementation varies by architecture, but these concepts provide the foundation for the patterns covered in the rest of this handbook.
