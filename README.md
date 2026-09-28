# AGENTIC AI SYSTEM DESIGN

A practical, framework-agnostic handbook for designing **production-grade agentic AI systems**.

The goal is not to learn how to call an LLM.

The goal is to answer:

> **Given a business problem, should I use a workflow, an agent, multiple agents, RAG, memory, tools, or some combination — and how do I design the system safely and reliably?**

---

# Table of Contents

1. [The Fundamental Mental Model](#1-the-fundamental-mental-model)
2. [Workflow vs Agent](#2-workflow-vs-agent)
3. [Agent Autonomy Must Have Boundaries](#3-agent-autonomy-must-have-boundaries)
4. [The Agent Loop](#4-the-agent-loop)
5. [Workflow Patterns](#5-workflow-patterns)
6. [Reasoning and Planning Patterns](#6-reasoning-and-planning-patterns)
7. [Multi-Agent Patterns](#7-multi-agent-patterns)
8. [RAG Architecture](#8-rag-architecture)
9. [Memory and State](#9-memory-and-state)
10. [Tools and Agent Communication](#10-tools-and-agent-communication)
11. [Reliability](#11-reliability)
12. [Security and Governance](#12-security-and-governance)
13. [Observability](#13-observability)
14. [Evaluation](#14-evaluation)
15. [Production Architecture Patterns](#15-production-architecture-patterns)
16. [The Most Important Architectural Principle](#16-the-most-important-architectural-principle)
17. [How to Design an Agentic System](#17-how-to-design-an-agentic-system)
18. [Universal Production Agent Architecture](#18-universal-production-agent-architecture)
19. [Decision Cheat Sheet](#19-decision-cheat-sheet)
20. [Final Mental Model](#20-final-mental-model)

---

# 1. The Fundamental Mental Model

An LLM is not automatically an agent.

```text
LLM
 ↓
Reasoning capability

Agent
 ↓
LLM + Context + State + Tools + Execution Loop

Production Agentic System
 ↓
Agent
+ Memory
+ Guardrails
+ Authorization
+ Reliability
+ Observability
+ Evaluation
+ Human Control
```

A useful mental model:

```text
THINK       → Model
CONTEXT     → Instructions + State + Memory
ACT         → Tools
EXECUTE     → Runtime + Orchestration
CONTROL     → Guardrails + Authorization + Human Approval
OBSERVE     → Tracing + Logging + Audit
EVALUATE    → Output + Trajectory + Tool Evaluation
```

---

# 2. Workflow vs Agent

This is the first architectural decision.

```text
Workflow
    ↓
Developer determines the execution path

Agent
    ↓
Model dynamically determines the execution path

Hybrid
    ↓
Developer defines boundaries
Agent dynamically decides within those boundaries
```

The key question is:

> **Is the next step known before execution begins?**

If yes:

```text
Workflow
```

If no, and the next step depends on observations:

```text
Agent
```

Do not introduce an agent merely because an LLM exists somewhere in the system.

---

# 3. Agent Autonomy Must Have Boundaries

An agent can make decisions, but the system must define the boundaries.

Four important autonomy boundaries are:

```text
Decision
Action
Resource
Termination
```

For example:

```text
Agent can decide:
Which research tool to use

System decides:
Whether the tool is authorized

Agent can decide:
What to investigate next

System decides:
Maximum iterations and cost
```

The core principle:

> **Let the agent decide within the boundary; let the system define the boundary.**

---

# 4. The Agent Loop

Most agentic systems can be understood as:

```text
GOAL
 ↓
DECIDE
 ↓
ACT
 ↓
OBSERVE
 ↓
DECIDE AGAIN
 ↓
...
 ↓
COMPLETE
```

The three fundamental operations are:

```text
Decide
Act
Observe
```

Everything else — planning, reflection, tool calling, RAG, multi-agent communication — builds on this loop.

---

# 5. Workflow Patterns

Use workflow patterns when the execution structure is known.

## Sequential

```text
A → B → C → D
```

Use when every step depends on the previous step.

## Parallel

```text
       ┌→ A ─┐
Input ─┼→ B ─┼→ Output
       └→ C ─┘
```

Use when tasks are independent.

The key question:

> **Does this task depend on the output of another task?**

## Router

```text
Input
 ↓
Classify
 ↓
Route
 ├── A
 ├── B
 └── C
```

Use when there are multiple predefined execution paths.

Routing is not authorization.

## Fan-Out / Fan-In

```text
          ┌→ Agent A ─┐
Input ────┼→ Agent B ─┼→ Combine
          └→ Agent C ─┘
```

Use when independent work can happen concurrently and results must later be combined.

## Conditional

```text
Condition
 ├── True  → A
 └── False → B
```

Use when a known business rule determines the path.

Business rules should generally remain outside the LLM.

## Loop

```text
Execute
   ↓
Condition
   ↓
Continue / Stop
```

Use when a known process must repeat until a termination condition is satisfied.

---

# 6. Reasoning and Planning Patterns

## ReAct

```text
Reason
 ↓
Act
 ↓
Observe
 ↓
Reason
```

Useful when the next action depends directly on the latest observation.

## Plan-and-Execute

```text
Goal
 ↓
Plan
 ↓
Execute
 ↓
Observe
 ↓
Re-plan if necessary
```

Useful when a complex task benefits from an explicit high-level plan.

## Reflection

```text
Generate
 ↓
Critique
 ↓
Improve
```

Useful when an existing result can be reviewed and improved.

## Evaluator-Optimizer

```text
Generator
    ↓
Result
    ↓
Evaluator
    ↓
Pass / Feedback
    ↓
Optimizer
```

The important distinction:

```text
Evaluator
    defines what "good" means

Optimizer
    tries to satisfy those criteria
```

---

# 7. Multi-Agent Patterns

Multiple agents should exist because there are meaningful boundaries between capabilities — not simply because "multi-agent" sounds more advanced.

## Supervisor

```text
Supervisor
 ├── Agent A
 ├── Agent B
 └── Agent C
```

Central coordinator.

## Hierarchical

```text
Global Supervisor
      ↓
Domain Supervisor
      ↓
Specialist Agent
```

Useful when one coordinator becomes too complex.

## Peer-to-Peer

```text
Agent A ↔ Agent B ↔ Agent C
```

Useful when agents need direct collaboration.

Requires strong communication and cycle controls.

## Agent-as-Tool

```text
Agent A
   ↓
Agent B
```

Agent B is exposed through a controlled capability interface.

## Subgraphs

```text
Parent Agent
    ↓
Subgraph
 ├── Step A
 ├── Step B
 └── Step C
```

A subgraph encapsulates a reusable workflow or agent system.

Mental model:

> **Parent owns what; subgraph owns how.**

---

# 8. RAG Architecture

Basic RAG:

```text
Query
 ↓
Retrieve
 ↓
Context
 ↓
Generate
```

But real production systems often need more.

## Agentic RAG

```text
Agent
 ↓
Decide what to retrieve
 ↓
Retrieve
 ↓
Observe
 ↓
Retrieve again if necessary
```

The difference:

```text
Basic RAG
Application controls retrieval

Agentic RAG
Agent controls retrieval decisions
```

## Corrective RAG

```text
Retrieve
 ↓
Evaluate
 ↓
Good? ── Yes → Generate
  │
  No
  ↓
Correct / Retrieve Again
```

Useful when retrieval quality is uncertain.

## Multi-Hop RAG

```text
Question
 ↓
Retrieve A
 ↓
Discover B
 ↓
Retrieve B
 ↓
Discover C
 ↓
Retrieve C
```

Use when one piece of information determines the next retrieval.

## Hybrid RAG

Combine:

```text
Vector Search
+
Keyword Search
+
Optional Reranking
```

Useful when both semantic meaning and exact terms matter.

---

# 9. Memory and State

Memory and state are different concepts.

## Short-Term Memory

What the agent needs during the current interaction.

```text
Conversation
Current task
Current context
```

## Long-Term Memory

Persistent information useful across future interactions.

```text
Preferences
Historical information
Persistent facts
```

## Episodic Memory

Remembering previous experiences.

```text
Situation
 ↓
Actions
 ↓
Outcome
```

Useful when the agent benefits from knowing what happened in similar situations.

## Semantic Memory

Remembering facts and relationships.

```text
Fact
Concept
Relationship
```

Independent of a specific previous experience.

## Durable State

Execution state that survives:

```text
Crash
Pause
Restart
Human approval
```

Durable state is about **execution continuity**, not necessarily memory.

---

# 10. Tools and Agent Communication

The LLM should not directly execute external operations.

The basic model is:

```text
LLM
 ↓
Tool Request
 ↓
Application
 ↓
Authorization / Policy
 ↓
Tool
```

A useful principle:

> **LLM decides WHAT, application decides WHETHER, tool determines HOW.**

## MCP

MCP standardizes how AI applications interact with external tools and context.

Useful when an agent needs a standardized ecosystem of external capabilities.

## A2A

A2A is for:

```text
Agent ↔ Agent
```

rather than:

```text
Agent → Simple Tool
```

Agents should communicate through explicit contracts rather than sharing unrestricted internal state.

## Tool Discovery

For large tool ecosystems:

```text
Registry
 ↓
Discover
 ↓
Authorize
 ↓
Select
 ↓
Execute
```

Discovery should respect permissions.

Unauthorized tools should not simply become available because the model discovered them.

---

# 11. Reliability

Agentic systems fail in more ways than ordinary applications.

Important patterns:

```text
Retry
Timeout
Fallback
Circuit Breaker
Idempotency
Checkpoint / Resume
DLQ
```

## Retry

Use for transient failures.

Always consider:

```text
Backoff
Jitter
Retry Budget
Idempotency
```

## Timeout

Every external operation needs a bounded execution time.

Timeouts should propagate through nested agent calls.

## Fallback

Use an alternative execution path when a dependency fails.

## Circuit Breaker

```text
CLOSED
 ↓ failures
OPEN
 ↓ cooldown
HALF-OPEN
 ↓ successful probe
CLOSED
```

Prevents repeatedly calling an unhealthy dependency.

## Idempotency

Retries can create duplicate side effects.

```text
Logical Operation
      ↓
Idempotency Key
      ↓
Safe Retry
```

Critical for:

```text
Payments
Refunds
Orders
Messages
External Actions
```

## Checkpoint / Resume

```text
Execute
 ↓
Checkpoint
 ↓
Crash / Pause
 ↓
Resume
```

Important for long-running agents.

## DLQ

```text
Message
 ↓
Retry
 ↓
Retry
 ↓
Retry exhausted
 ↓
DLQ
```

A DLQ should be observable and replayable.

---

# 12. Security and Governance

Security must exist **outside the LLM**.

## Authentication vs Authorization

```text
Authentication
    ↓
Who are you?

Authorization
    ↓
What are you allowed to do?
```

The agent should never be its own authorization system.

## Tenant Isolation

```text
Identity
 ↓
Tenant Context
 ↓
Agent
 ↓
Memory / RAG / DB / Tools
```

Never trust a tenant ID supplied by a prompt.

## Prompt Injection

External content is untrusted:

```text
User
Web
Email
Documents
Tool Results
```

Treat these as data, not trusted instructions.

A prompt injection must never bypass:

```text
Authentication
Authorization
Policy
Tool Permissions
```

## Tool Permissions

Permissions can depend on:

```text
Agent
Action
Resource
Tenant
Environment
Time
Context
```

The architecture should be:

```text
LLM
 ↓
Tool Request
 ↓
Permission System
 ↓
Tool
```

## Human Approval

Use human approval for high-risk actions.

```text
Agent
 ↓
Proposed Action
 ↓
Policy
 ↓
Human Approval
 ↓
Execute
```

Approval should apply to the actual action being performed.

---

# 13. Observability

Agentic systems require more than ordinary logs.

## Agent Tracing

A trace represents the complete logical execution.

```text
Agent
 ├── LLM
 ├── Tool
 ├── Retrieval
 ├── Agent
 └── Subgraph
```

## Tool Tracing

Track external operations separately.

```text
LLM Decision
    ↓
Tool Span
    ↓
External Operation
```

## Cost

Track cost at:

```text
Request
Tenant
Agent
Model
Tool
```

Control with:

```text
MAX_LLM_CALLS
MAX_TOKENS
MAX_TOOL_CALLS
MAX_COST
MAX_ITERATIONS
```

## Latency

Optimize the critical path.

Sequential operations increase latency:

```text
A → B → C → D
```

Independent operations can run concurrently:

```text
A ─┐
B ─┼→ C
D ─┘
```

Track:

```text
TTFT
Time to useful result
Final latency
p50
p95
p99
```

## Audit

Observability asks:

> What happened?

Audit asks:

> Who performed what action, on which resource, under what authorization?

Audit is particularly important for:

```text
Authentication
Authorization
High-risk actions
Human approval
External side effects
```

---

# 14. Evaluation

A successful execution does not necessarily mean a good agent.

Evaluation should happen at multiple levels.

## Final Output Evaluation

```text
Was the result correct?
Complete?
Relevant?
Grounded?
Safe?
```

## Trajectory Evaluation

```text
Did the agent take the right path?
```

Evaluate:

```text
Tool selection
Tool arguments
Order of actions
Recovery
Efficiency
Termination
Policy compliance
```

## Tool Evaluation

```text
Did the agent choose the right tool?
Were the arguments correct?
Did it interpret the result correctly?
```

## Agent Benchmarks

Use standardized task datasets to measure agent behavior repeatedly.

```text
Dataset
 ↓
Agent
 ↓
Execution
 ↓
Evaluation
 ↓
Metrics
```

Useful for regression testing across:

```text
Model versions
Agent versions
Prompt versions
Tool changes
Architecture changes
```

---

# 15. Production Architecture Patterns

The final chapter applies the previous patterns to complete systems.

## Research Agent

A research agent is useful when:

```text
The required information is unknown
        ↓
Research path is adaptive
        ↓
Evidence must be evaluated
        ↓
Additional research may be required
```

Core loop:

```text
PLAN
 ↓
SELECT
 ↓
RESEARCH
 ↓
EVALUATE
 ↓
Research Again / Complete
 ↓
REPORT
```

A key lesson:

> One impossible research question should not necessarily stop the entire research task.

Research questions can be exhausted independently.

## Coding Agent

A coding agent operates inside a controlled software-development environment.

```text
PLAN
 ↓
INSPECT
 ↓
IMPLEMENT
 ↓
TEST
 ↓
EVALUATE
 │
 ├── Pass → REVIEW
 │
 └── Fail → DIAGNOSE → FIX → TEST
```

The agent reasons about implementation.

The runtime controls:

```text
Repository permissions
Execution sandbox
Resource limits
Production deployment
Approval
```

## Sales Agent

A sales platform should not make everything an agent.

Use:

```text
Deterministic Pipeline
    ↓
Account Discovery
    ↓
Feature Extraction
    ↓
Signal Detection
    ↓
Initial Scoring
```

Then use an agent when:

```text
Evidence is incomplete
        ↓
Research path is unknown
        ↓
Agent investigates dynamically
        ↓
Validated evidence
```

Then:

```text
Validated Evidence
 ↓
Scoring
 ↓
Outreach Agent
 ↓
Policy / Approval
 ↓
External Action
```

Key principle:

> **Do not make something an agent simply because an LLM can perform it.**

## Customer Support Agent

A support agent is useful when diagnosis requires multiple systems and the next action depends on previous observations.

```text
CUSTOMER PROBLEM
 ↓
UNDERSTAND
 ↓
SELECT TOOL
 ↓
OBSERVE
 ↓
EVALUATE
 │
 ├── Enough → Resolve
 │
 └── Missing → Investigate Again
```

For state-changing operations:

```text
Agent
 ↓
Proposed Action
 ↓
Authorization
 ↓
Business Policy
 ↓
Execute
 ↓
Verify
```

The agent should never replace transactional business systems.

## Multi-Agent Platform

A multi-agent platform is infrastructure for safely running many specialized agents.

```text
Client
 ↓
Gateway
 ↓
Identity
 ↓
Agent Router
 ↓
Agent Runtime
 ↓
Specialized Agents
 ↓
Tool Gateway
 ↓
External Systems
```

Platform responsibilities:

```text
Execution
Identity
Isolation
Agent Registry
Versioning
Tool Access
Checkpoints
Observability
Cost
Governance
```

Agent responsibilities:

```text
Domain reasoning
Planning
Decision-making
Tool selection
Domain-specific evaluation
```

---

# 16. The Most Important Architectural Principle

Across the entire handbook, the same principle appears repeatedly:

> **Use deterministic systems for deterministic work. Use agents where the execution path is genuinely uncertain and must be determined from observations.**

For example:

```text
Known business rule
      ↓
Workflow

Known data transformation
      ↓
Pipeline

Known database query
      ↓
Service

Unknown research path
      ↓
Agent

Dynamic tool selection
      ↓
Agent

Adaptive investigation
      ↓
Agent

High-risk action
      ↓
Policy / Authorization / Human Approval
```

---

# 17. How to Design an Agentic System

When given a new business problem, work through this sequence.

## Step 1 — Define the Goal

What is the system actually trying to accomplish?

```text
Goal
```

## Step 2 — Identify Known vs Unknown Execution

Ask:

> Can I define the execution path before runtime?

If yes:

```text
Workflow
```

If no:

```text
Agent
```

## Step 3 — Identify Tools

What external capabilities are required?

```text
Database
API
Search
Code execution
CRM
Payment
Cloud
```

## Step 4 — Define State

What must survive between steps?

```text
Current task
Observations
Tool results
Progress
Failures
Approvals
```

## Step 5 — Define Autonomy Boundaries

What can the agent decide?

What can it never decide?

```text
Decision boundary
Action boundary
Resource boundary
Termination boundary
```

## Step 6 — Define Failure Handling

For every external dependency ask:

```text
What if it is slow?
What if it fails?
What if the response is duplicated?
What if the agent crashes?
What if the same action is retried?
```

Map the answer to:

```text
Timeout
Retry
Fallback
Circuit Breaker
Idempotency
Checkpoint
DLQ
```

## Step 7 — Define Security

Before exposing tools, determine:

```text
Who is the user?
What tenant are they in?
What can the agent access?
What can it modify?
What requires approval?
```

## Step 8 — Define Observability

You should be able to answer:

```text
What did the agent do?
Why did it do it?
Which tools were called?
What did they return?
How much did it cost?
How long did it take?
Why did it stop?
```

## Step 9 — Define Evaluation

Measure both:

```text
Final Result
+
Trajectory
```

Do not evaluate only the final answer.

---

# 18. Universal Production Agent Architecture

A mature production system often looks like:

```text
                         USER
                           │
                           ↓
                     API GATEWAY
                           │
                           ↓
                 AUTHENTICATION / TENANT
                           │
                           ↓
                     ORCHESTRATOR
                           │
                           ↓
                       AGENT
                           │
               ┌───────────┼───────────┐
               ↓           ↓           ↓
             STATE       MEMORY       TOOLS
               │                       │
               │                  TOOL GATEWAY
               │                       │
               │               AUTH / POLICY
               │                       │
               │                       ↓
               │                EXTERNAL SYSTEMS
               │
               ↓
          CHECKPOINTS

               ┌──────────────────────────┐
               │ Cross-Cutting Controls   │
               │                          │
               │ Reliability              │
               │ Observability            │
               │ Cost                     │
               │ Evaluation               │
               │ Audit                    │
               │ Human Approval           │
               └──────────────────────────┘
```

This is not a mandatory architecture.

It is a **mental model for decomposing production agentic systems**.

---

# 19. Decision Cheat Sheet

| Problem                               | Default Architecture       |
| ------------------------------------- | -------------------------- |
| Fixed sequence                        | Sequential workflow        |
| Independent tasks                     | Parallel                   |
| Multiple predefined paths             | Router                     |
| Known business condition              | Conditional                |
| Repeated known process                | Loop                       |
| Dynamic tool selection                | Agent / ReAct              |
| Complex upfront planning              | Plan-and-Execute           |
| Improve an existing result            | Reflection                 |
| Generate → judge → improve            | Evaluator-Optimizer        |
| Multiple specialists with coordinator | Supervisor                 |
| Deep organizational hierarchy         | Hierarchical agents        |
| Direct agent collaboration            | Peer-to-Peer / A2A         |
| Agent encapsulated as capability      | Agent-as-Tool              |
| Reusable agent/workflow component     | Subgraph                   |
| Simple document retrieval             | Basic RAG                  |
| Agent chooses retrieval strategy      | Agentic RAG                |
| Retrieval quality uncertain           | Corrective RAG             |
| Dependent retrieval chain             | Multi-Hop RAG              |
| Semantic + exact matching             | Hybrid RAG                 |
| Current conversation context          | Short-Term Memory          |
| Persistent user knowledge             | Long-Term Memory           |
| Previous experiences                  | Episodic Memory            |
| Persistent facts                      | Semantic Memory            |
| Crash-resistant execution             | Durable State / Checkpoint |
| External capability                   | Tool Calling               |
| Standardized external tools/context   | MCP                        |
| Agent ↔ Agent                         | A2A                        |
| Huge tool ecosystem                   | Tool Discovery             |
| Transient failure                     | Retry                      |
| Slow dependency                       | Timeout                    |
| Alternative path                      | Fallback                   |
| Repeated dependency failure           | Circuit Breaker            |
| Duplicate side effects                | Idempotency                |
| Long-running execution                | Checkpoint / Resume        |
| Permanently failed messages           | DLQ                        |
| Identity                              | Authentication             |
| Access control                        | Authorization              |
| Multi-tenant system                   | Tenant Isolation           |
| Untrusted instructions in content     | Prompt Injection Defense   |
| Restrict agent actions                | Tool Permissions           |
| Risky operation                       | Human Approval             |
| Understand execution                  | Agent Tracing              |
| Understand external calls             | Tool Tracing               |
| Control spend                         | Cost Controls              |
| Optimize response time                | Latency                    |
| Accountability                        | Audit                      |
| Evaluate result                       | Final Output Evaluation    |
| Evaluate agent path                   | Trajectory Evaluation      |
| Evaluate tool usage                   | Tool Evaluation            |
| Repeatable measurement                | Agent Benchmarks           |

---

# 20. Final Mental Model

When designing an agentic system, think in this order:

```text
                    BUSINESS GOAL
                          ↓
               Is the path predictable?
                    /           \
                  YES            NO
                   ↓              ↓
               WORKFLOW         AGENT
                   │              │
                   └──────┬───────┘
                          ↓
                        STATE
                          ↓
                        TOOLS
                          ↓
                 AUTHORIZATION / POLICY
                          ↓
                     EXECUTION
                          ↓
                OBSERVE / EVALUATE
                          ↓
                  CONTINUE / STOP
                          ↓
                 RELIABILITY CONTROLS
                          ↓
                 OBSERVABILITY / AUDIT
                          ↓
                     EVALUATION
```

And for every agent:

```text
WHAT IS THE GOAL?
        ↓
WHAT DOES THE AGENT DECIDE?
        ↓
WHAT DOES THE SYSTEM DECIDE?
        ↓
WHAT TOOLS CAN IT USE?
        ↓
WHAT STATE MUST SURVIVE?
        ↓
WHAT CAN GO WRONG?
        ↓
WHAT ACTIONS REQUIRE AUTHORIZATION?
        ↓
HOW DO WE OBSERVE IT?
        ↓
HOW DO WE KNOW IT WORKED?
```

## Final Principle

> **Good agentic system design is not about maximizing autonomy. It is about giving an agent exactly the autonomy required to solve the uncertain parts of a problem, while deterministic systems, policies, security boundaries, and runtime controls govern everything else.**
