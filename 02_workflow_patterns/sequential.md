# Sequential Workflow

## 1. Problem

Many business processes naturally follow a fixed sequence of steps.

For example:

```text
Fetch Data
    ↓
Clean Data
    ↓
Analyze Data
    ↓
Generate Report
```

The next step depends on the output of the previous step.

In these situations, there is usually no reason for an agent to dynamically decide what happens next.

A deterministic workflow is often simpler, easier to test, and easier to operate.

---

# 2. Core Idea

A **Sequential Workflow** executes a predefined sequence of steps where:

```text
Step 1
  ↓
Step 2
  ↓
Step 3
  ↓
Step 4
```

Each step:

1. Receives input
2. Performs its task
3. Produces output
4. Passes that output to the next step

The application, rather than the model, controls the execution order.

### Mental Model

```text
Developer defines:

A → B → C → D

Runtime executes:

A
 ↓
B
 ↓
C
 ↓
D
```

The model may be used **inside individual steps**, but it does not necessarily control the overall workflow.

---

# 3. Architecture

A basic sequential workflow looks like:

```text
                    ┌─────────────┐
                    │   Input     │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │   Step A    │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │   Step B    │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │   Step C    │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │   Step D    │
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │   Output    │
                    └─────────────┘
```

The important property is that the **execution graph is known before runtime**.

---

# 4. Execution Flow

Suppose we are building an AI research pipeline:

```text
User Query
    ↓
Retrieve Documents
    ↓
Extract Relevant Information
    ↓
Analyze Information
    ↓
Generate Summary
```

Execution:

```text
query
  │
  ▼
retrieve(query)
  │
  │ documents
  ▼
extract(documents)
  │
  │ relevant_information
  ▼
analyze(relevant_information)
  │
  │ analysis
  ▼
summarize(analysis)
  │
  ▼
final_answer
```

The workflow does not ask:

> "What should I do next?"

Instead, the workflow already knows:

```text
retrieve → extract → analyze → summarize
```

---

# 5. Data Flow Between Steps

A sequential workflow becomes especially useful when one step depends on the previous step's output.

Example:

```text
Step 1
Input:
    user_query

Output:
    documents


Step 2
Input:
    documents

Output:
    relevant_chunks


Step 3
Input:
    relevant_chunks

Output:
    analysis


Step 4
Input:
    analysis

Output:
    final_answer
```

This creates a pipeline:

```text
Input
  ↓
Transformation
  ↓
Transformation
  ↓
Transformation
  ↓
Output
```

---

# 6. Example: Customer Support

Imagine an automated customer-support workflow.

Requirement:

> When a customer submits a support ticket, classify it, retrieve relevant knowledge, generate a response, and store the response.

The workflow could be:

```text
Ticket
  ↓
Classify Issue
  ↓
Retrieve Knowledge
  ↓
Generate Response
  ↓
Validate Response
  ↓
Store Response
```

Each step has a specific responsibility.

### Step 1 — Classification

```text
Input:
Customer ticket

Output:
category = "billing"
```

### Step 2 — Retrieval

```text
Input:
category + ticket

Output:
relevant knowledge
```

### Step 3 — Generation

```text
Input:
ticket + knowledge

Output:
draft response
```

### Step 4 — Validation

```text
Input:
draft response

Output:
validated response
```

### Step 5 — Storage

```text
Input:
validated response

Output:
stored ticket response
```

The workflow is deterministic:

```text
classify
   ↓
retrieve
   ↓
generate
   ↓
validate
   ↓
store
```

---

# 7. Sequential Workflow With LLMs

Sequential does **not** mean the workflow cannot use agents or LLMs.

For example:

```text
User Request
     ↓
LLM: Extract Requirements
     ↓
Code Tool
     ↓
LLM: Analyze Result
     ↓
LLM: Generate Explanation
```

The execution order is still deterministic.

The LLM provides intelligence **inside the workflow**, while the workflow controls the overall execution.

This distinction is important:

```text
LLM inside workflow
        ≠
LLM controlling workflow
```

---

# 8. Sequential Workflow vs Agent

Consider:

```text
Retrieve
  ↓
Analyze
  ↓
Summarize
```

### Sequential Workflow

The developer defines:

```text
Retrieve → Analyze → Summarize
```

### Agent

The agent may decide:

```text
Retrieve
   ↓
Analyze
   ↓
Need more information?
   ├── Yes → Search again
   │          ↓
   │       Analyze
   │
   └── No → Summarize
```

The difference is not whether an LLM exists.

The difference is **who controls the next step**.

```text
Workflow:
Developer → controls execution

Agent:
Model → dynamically influences execution
```

---

# 9. When to Use Sequential Workflow

Use this pattern when:

### 1. The process has a known order

```text
A → B → C → D
```

### 2. Each step depends on the previous step

For example:

```text
Parse
 ↓
Validate
 ↓
Transform
 ↓
Persist
```

### 3. Predictability is important

You want the same execution structure for every request.

### 4. The process has strong business rules

For example:

```text
Validate payment
 ↓
Check fraud
 ↓
Authorize
 ↓
Capture payment
```

The business process should not depend on an LLM deciding which step to execute.

### 5. You need easy debugging

A failed execution can be traced to:

```text
Step 1 ✓
Step 2 ✓
Step 3 ✗
Step 4 not executed
```

---

# 10. When NOT to Use Sequential Workflow

Do not force a sequential workflow when the next action depends heavily on runtime reasoning.

For example:

> "Research this company and find everything relevant to its cybersecurity posture."

The system may need to dynamically decide:

```text
Search web
   ↓
Found vulnerability?
   ├── Yes → investigate CVE
   └── No  → search infrastructure

Need more evidence?
   ├── Yes → search again
   └── No  → generate report
```

A rigid pipeline becomes difficult because the execution path is not known beforehand.

This is where patterns such as:

* Router
* Conditional
* Loop
* ReAct
* Plan-and-Execute
* Agent Loop

become more appropriate.

---

# 11. Sequential Workflow With Failure Handling

A production workflow should define what happens when a step fails.

Basic version:

```text
A
↓
B
↓
C
↓
D
```

If `C` fails:

```text
A ✓
↓
B ✓
↓
C ✗
↓
STOP
```

But production systems may implement:

```text
C
↓
Failure?
 ├── Retry
 ├── Fallback
 ├── Skip
 ├── Compensate
 └── Fail workflow
```

For example:

```text
Retrieve Data
     ↓
API Failure
     ↓
Retry
     ↓
Still failing?
     ├── Yes → Fallback source
     └── No  → Continue
```

The actual reliability mechanisms belong to Chapter 8, but they should be considered when designing the workflow.

---

# 12. State in Sequential Workflows

A sequential workflow usually maintains state between steps.

Example:

```text
state = {
    "query": "...",
    "documents": [...],
    "analysis": "...",
    "final_answer": "..."
}
```

Execution:

```text
Step A
   ↓
state.documents = [...]

Step B
   ↓
state.analysis = ...

Step C
   ↓
state.final_answer = ...
```

The state provides the shared execution context.

This becomes particularly important when workflows need:

* Checkpointing
* Resume after failure
* Human approval
* Auditing
* Long-running execution

These topics will be covered later.

---

# 13. Sequential Workflow With Human Approval

Sequential workflows can also contain human-in-the-loop steps.

Example:

```text
Generate Proposal
       ↓
Human Review
       ↓
Approved?
   ├── No → Revise
   └── Yes
       ↓
Publish
```

The workflow remains sequential, but one step pauses execution.

This is common for high-impact actions where the system should prepare the work automatically but require explicit approval before execution.

---

# 14. Sequential Workflow vs Sequential Agent

These terms can sound similar but represent different concepts.

### Sequential Workflow

```text
A → B → C → D
```

The application controls the sequence.

### Sequential Agents

```text
Agent A
   ↓
Agent B
   ↓
Agent C
```

The workflow is still sequential, but each step may itself be an autonomous agent.

For example:

```text
Research Agent
      ↓
Analysis Agent
      ↓
Writing Agent
```

The overall orchestration remains deterministic:

```text
Research → Analysis → Writing
```

Even though each component may internally use agentic reasoning.

---

# 15. Trade-offs

| Aspect                    | Sequential Workflow        |
| ------------------------- | -------------------------- |
| Execution order           | Fixed                      |
| Predictability            | High                       |
| Debugging                 | Easy                       |
| Testing                   | Relatively easy            |
| Control                   | High                       |
| Flexibility               | Limited                    |
| Runtime decisions         | Limited                    |
| Cost control              | Easier                     |
| Failure isolation         | Easier                     |
| Implementation complexity | Low                        |
| Best for                  | Known multi-step processes |

The main trade-off is:

```text
More control
    ↓
Less flexibility
```

---

# 16. Production Considerations

For production systems, think beyond the happy path.

### Idempotency

If a workflow retries a step, it should not accidentally perform the same side effect twice.

Example:

```text
Charge Customer
```

must be idempotent.

---

### Timeouts

Every external call should have a bounded execution time.

```text
Step
 ↓
Timeout
 ↓
Retry / Fallback / Fail
```

---

### Checkpointing

For long-running workflows:

```text
A ✓
↓
B ✓
↓
Checkpoint
↓
C ✗
```

The workflow should ideally resume from `C` rather than restarting from `A`.

---

### Observability

Track:

```text
workflow_id
run_id
step_id
input
output
latency
error
retry_count
model
token_usage
cost
```

This allows operators to answer:

> "Where did this workflow fail?"

---

### Auditability

For important workflows, record:

```text
Who started it?
What input was provided?
Which steps executed?
What decisions were made?
What tools were called?
What was the final result?
```

---

# 17. Example Architecture

A production AI document-processing pipeline might look like:

```text
                ┌─────────────────┐
                │  Document Upload│
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │ Parse Document  │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │ Extract Metadata│
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │ Generate Summary│
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │ Validate Output │
                └────────┬────────┘
                         ↓
                ┌─────────────────┐
                │ Store Result    │
                └─────────────────┘
```

This is a strong candidate for a sequential workflow because the process has a predictable order.

---

# 18. Interview Question

### Question

> "You need to build an AI pipeline that extracts information from a document, validates it, summarizes it, and stores the result. Would you use an agent?"

A good architectural answer is:

> "I would start with a sequential workflow because the processing stages and their dependencies are known. I would use LLMs inside individual steps where reasoning is useful, but keep orchestration deterministic. If requirements later introduce dynamic decisions, such as deciding which analysis steps are required based on document content, I would introduce conditional routing or agentic orchestration only where necessary."

This demonstrates an important architectural principle:

> **Do not introduce agent autonomy where deterministic orchestration is sufficient.**

Structured workflows are often useful precisely because they make predictable parts of an AI system explicit and controllable.

---

# 19. Decision Rule

When evaluating a new system, ask:

```text
Is the execution path known beforehand?
            │
       ┌────┴────┐
      Yes        No
       │          │
       ▼          ▼
 Sequential    Dynamic
 Workflow      Orchestration
```

If:

```text
A → B → C → D
```

is always the expected process, use a sequential workflow.

If the process looks like:

```text
A
 ↓
What should I do next?
 ↓
B or C or D?
 ↓
Need more information?
 ↓
Search again?
```

then sequential execution alone may not be enough.

---

# 20. Most Important Mental Model

Remember:

```text
Sequential Workflow

Developer defines:
    A → B → C → D

Runtime executes:
    A
    ↓
    B
    ↓
    C
    ↓
    D
```

The model can provide intelligence inside each step:

```text
Workflow
   │
   ├── LLM Step
   ├── Tool Step
   ├── Code Step
   ├── LLM Step
   └── Database Step
```

But the workflow itself controls **what happens next**.

### Core Principle

> **Use deterministic orchestration for deterministic processes.**

Agentic behavior should be introduced when the problem actually requires runtime decision-making.

---

# 21. Relationship to Other Workflow Patterns

Sequential is the baseline.

From here, more complex patterns emerge:

```text
Sequential
    │
    ├── Need independent tasks at the same time?
    │        ↓
    │     Parallel
    │
    ├── Need to choose one path?
    │        ↓
    │     Router
    │
    ├── Need multiple branches + combine results?
    │        ↓
    │     Fan-Out / Fan-In
    │
    ├── Need runtime branching?
    │        ↓
    │     Conditional
    │
    └── Need repeated execution?
             ↓
           Loop
```

This makes Sequential Workflow the foundation for understanding the rest of Chapter 2.
