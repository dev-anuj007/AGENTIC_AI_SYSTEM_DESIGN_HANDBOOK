# Coding Agent

## Problem

A coding agent must do more than generate code.

It needs to understand a software task, inspect an existing codebase, modify files, run tests, interpret failures, and potentially create a pull request.

Unlike a simple code-generation workflow, the next action often depends on what the agent discovers during execution.

A typical flow is:

```text
User Request
     ↓
Understand Task
     ↓
Inspect Repository
     ↓
Plan
     ↓
Implement
     ↓
Test
     ↓
Evaluate
     │
     ├── Pass → Review / PR
     │
     └── Fail → Diagnose → Fix → Test Again
```

---

# Core Architecture

A production coding agent can be structured as:

```text
User Goal
    ↓
Coding Agent
    ↓
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
    ├── Pass
    │     ↓
    │   REVIEW
    │     ↓
    │   PR / RESULT
    │
    └── Fail
          ↓
       DIAGNOSE
          ↓
        FIX
          ↓
        TEST
```

The important design principle is:

> The agent can reason about the implementation, but the runtime controls what it is allowed to change and execute.

---

# Example

Suppose the user asks:

> Fix the authentication bug in the payment service.

The agent should not immediately modify code.

It should first establish enough context:

```text
1. Inspect repository
2. Locate authentication code
3. Identify relevant tests
4. Understand failure
5. Create implementation plan
6. Modify code
7. Run targeted tests
8. Run broader tests
9. Review diff
10. Create PR if authorized
```

The exact path can change based on what the agent discovers.

---

# State Design

The coding agent needs both **input/configuration state** and **runtime state**.

## Input / Configuration

The caller provides:

```text
goal
repository
branch
environment
max_iterations
max_test_attempts
```

## Runtime State

The agent manages:

```text
plan
files_inspected
files_modified
test_results
current_error
iteration_count
approval_status
pull_request
final_result
```

A simple state definition:

```python
from typing import TypedDict


class CodingState(TypedDict):
    # Input / configuration
    goal: str
    repository: str
    branch: str

    max_iterations: int
    max_test_attempts: int

    # Runtime state
    plan: list[str]

    files_inspected: list[str]
    files_modified: list[str]

    test_results: list[dict]
    current_error: str | None

    iteration_count: int
    test_attempts: int

    approval_status: str
    pull_request: str | None

    final_result: str
```

The caller should initialize the execution state explicitly:

```python
initial_state: CodingState = {
    "goal": "Fix authentication bug in payment service",
    "repository": "payment-service",
    "branch": "feature/auth-fix",

    "max_iterations": 10,
    "max_test_attempts": 3,

    "plan": [],

    "files_inspected": [],
    "files_modified": [],

    "test_results": [],
    "current_error": None,

    "iteration_count": 0,
    "test_attempts": 0,

    "approval_status": "not_required",
    "pull_request": None,

    "final_result": "",
}
```

---

# Execution Flow

## 1. Plan

The agent converts the user goal into implementation tasks.

```text
Goal
 ↓
Understand requirements
 ↓
Create implementation plan
```

Example:

```text
1. Inspect authentication middleware
2. Reproduce failure
3. Identify root cause
4. Implement fix
5. Add regression test
6. Run tests
7. Review diff
```

The plan should remain editable because repository discovery may invalidate assumptions.

---

# 2. Inspect

The agent uses read-only capabilities first.

Typical tools:

```text
list_files
read_file
search_code
read_git_history
inspect_dependencies
```

The agent should understand:

* Existing architecture
* Relevant modules
* Tests
* Dependencies
* Configuration
* Existing conventions

Read-only operations are generally lower risk than write or execution operations.

---

# 3. Implement

After enough context has been collected, the agent modifies the required files.

Typical capabilities:

```text
create_file
edit_file
delete_file
apply_patch
```

The tool layer should enforce which files and operations are allowed.

For example:

```text
Repository read       → allowed
Repository write      → allowed
Delete production config → denied
```

The model should never be the only security boundary.

---

# 4. Test

The agent executes tests in a controlled environment.

```text
Code
 ↓
Test Runner
 ↓
Results
```

Tests should preferably execute inside a sandbox with:

* CPU limits
* Memory limits
* Execution timeout
* Network restrictions
* Filesystem restrictions

For example:

```python
test_result = run_tests(
    command="pytest",
    timeout=120,
)
```

The runtime owns the timeout and resource controls.

---

# 5. Evaluate

The agent evaluates whether the implementation actually solves the problem.

Evaluation can combine deterministic checks:

```text
Tests
Lint
Type checking
Security scanning
Build
```

with semantic evaluation:

```text
Does the implementation satisfy the requirement?
```

The final decision should not depend solely on an LLM-generated statement such as:

```text
"Tests look good."
```

The system should inspect actual test results.

---

# Failure Loop

If tests fail:

```text
TEST
 ↓
Failure
 ↓
DIAGNOSE
 ↓
FIX
 ↓
TEST
```

Example:

```text
pytest
 ↓
3 failures
 ↓
Inspect stack trace
 ↓
Identify incorrect token validation
 ↓
Modify implementation
 ↓
pytest
 ↓
All tests pass
```

The loop must be bounded.

```text
MAX_ITERATIONS
MAX_TEST_ATTEMPTS
MAX_EXECUTION_TIME
MAX_COST
```

If the limit is reached, the agent should stop and report the remaining failure instead of endlessly modifying code.

---

# LangGraph Structure

A practical graph can be:

```text
START
  ↓
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
  │             ↓
  │           RESULT
  │
  └── Fail → DIAGNOSE
                ↓
               FIX
                ↓
               TEST
```

Example graph:

```python
from langgraph.graph import StateGraph, START, END


def plan(state: CodingState):
    return {
        "plan": [
            "Inspect authentication code",
            "Identify root cause",
            "Implement fix",
            "Add regression test",
            "Run tests",
        ]
    }


def inspect(state: CodingState):
    # Read-only repository inspection.
    return {
        "files_inspected": [
            "src/auth.py",
            "tests/test_auth.py",
        ]
    }


def implement(state: CodingState):
    # Actual implementation would use a controlled
    # repository editing tool.
    return {
        "files_modified": [
            "src/auth.py",
            "tests/test_auth.py",
        ]
    }


def test(state: CodingState):
    # Actual implementation would invoke a sandboxed runner.
    return {
        "test_attempts": state["test_attempts"] + 1,
        "test_results": [
            {
                "passed": True,
                "failures": [],
            }
        ],
    }


def evaluate(state: CodingState):
    results = state["test_results"]

    passed = bool(results) and all(
        result["passed"]
        for result in results
    )

    return {
        "current_error": None if passed else "Tests failed"
    }


def route_after_evaluation(state: CodingState):
    if state["current_error"] is None:
        return "review"

    if state["test_attempts"] >= state["max_test_attempts"]:
        return "result"

    return "diagnose"


def diagnose(state: CodingState):
    return {}


def fix(state: CodingState):
    return {}


def review(state: CodingState):
    return {}


def result(state: CodingState):
    return {
        "final_result": "Coding task completed or stopped with reported limitations."
    }


builder = StateGraph(CodingState)

builder.add_node("plan", plan)
builder.add_node("inspect", inspect)
builder.add_node("implement", implement)
builder.add_node("test", test)
builder.add_node("evaluate", evaluate)
builder.add_node("diagnose", diagnose)
builder.add_node("fix", fix)
builder.add_node("review", review)
builder.add_node("result", result)

builder.add_edge(START, "plan")
builder.add_edge("plan", "inspect")
builder.add_edge("inspect", "implement")
builder.add_edge("implement", "test")
builder.add_edge("test", "evaluate")

builder.add_conditional_edges(
    "evaluate",
    route_after_evaluation,
    {
        "review": "review",
        "diagnose": "diagnose",
        "result": "result",
    },
)

builder.add_edge("diagnose", "fix")
builder.add_edge("fix", "test")

builder.add_edge("review", "result")
builder.add_edge("result", END)

graph = builder.compile()
```

The code above illustrates the **architecture**, while repository editing and test execution should be implemented through controlled tools.

---

# Security Boundaries

A coding agent is especially sensitive because it can execute arbitrary code and modify software.

Separate capabilities:

```text
READ
 ├─ repository
 ├─ files
 └─ git history

WRITE
 ├─ files
 └─ branches

EXECUTE
 ├─ tests
 ├─ builds
 └─ scripts

PUBLISH
 ├─ pull request
 └─ release

DEPLOY
 ├─ staging
 └─ production
```

Each capability should have its own authorization policy.

For example:

```text
Read repository
    → allowed

Modify source code
    → allowed

Run tests
    → allowed in sandbox

Create PR
    → allowed

Merge PR
    → approval required

Production deployment
    → approval required
```

---

# Human Approval

High-impact operations should pause for approval.

For example:

```text
Agent
 ↓
Create deployment plan
 ↓
Permission check
 ↓
Human approval
 ↓
Production deployment
```

The approval should apply to the exact action and resource.

If the agent changes the deployment target or action after approval, the modified action should require a new authorization decision.

---

# Reliability

A production coding agent should use:

### Retry

For transient infrastructure failures.

```text
Git service timeout
 → retry
```

### Timeout

For test execution and external services.

```text
Test timeout
 → terminate process
```

### Checkpoint

For long-running coding tasks.

```text
Inspect
 ↓
Checkpoint
 ↓
Implement
 ↓
Checkpoint
 ↓
Test
```

### Idempotency

For operations such as PR creation or external task submission.

### Circuit breaker / fallback

For external services that repeatedly fail.

---

# Observability

A coding-agent trace should expose the execution hierarchy:

```text
Coding Agent
 ├─ Plan
 ├─ Repository Search
 ├─ File Read
 ├─ File Edit
 ├─ Test Execution
 ├─ Diagnosis
 ├─ Fix
 ├─ Test
 └─ Review
```

Track:

```text
request_id
trace_id
agent_id
repository
branch
tool
tool_latency
test_duration
token_usage
cost
iterations
failures
```

Do not expose secrets or sensitive repository content unnecessarily.

---

# Evaluation

A coding agent should be evaluated at two levels.

## Final Output

Did the implementation solve the task?

Measure:

* Tests passed
* Build passed
* Requirements satisfied
* Regression tests added where appropriate
* Security checks passed

## Trajectory

Did the agent behave correctly?

Measure:

* Appropriate tool selection
* Minimal unnecessary file access
* Correct tool arguments
* Permission compliance
* Efficient debugging
* Appropriate termination
* Recovery from failures

A coding agent that eventually produces correct code but repeatedly edits unrelated files has a trajectory problem.

---

# When to Use

Use a coding agent when:

* The task requires repository understanding
* The implementation path depends on discovered code
* Tests and iterative debugging are required
* Multiple development tools are needed
* The task benefits from autonomous execution

Do not use a fully autonomous coding agent when:

* The change is a simple deterministic transformation
* The operation is highly destructive without adequate controls
* A normal CI/CD workflow can solve the problem more reliably

---

# Trade-offs

| Benefit                           | Cost                        |
| --------------------------------- | --------------------------- |
| Autonomous repository exploration | Higher complexity           |
| Iterative debugging               | Higher execution cost       |
| Can adapt to unfamiliar code      | Harder to predict           |
| Can run tests and fix failures    | Requires sandboxing         |
| Can handle long tasks             | Requires checkpointing      |
| Can integrate many tools          | Requires strong permissions |

---

# Mental Model

```text
USER GOAL
    ↓
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
    ├── PASS
    │    ↓
    │  REVIEW
    │    ↓
    │  RESULT
    │
    └── FAIL
         ↓
      DIAGNOSE
         ↓
        FIX
         ↓
        TEST
```

> **A coding agent is an autonomous software-development loop operating inside a controlled repository, execution, security, and deployment boundary.**
