# Research Agent

## Problem

A research agent must gather information from multiple sources, identify missing information, validate evidence, and produce a grounded result.

A basic RAG flow:

```text
Query
 ↓
Retrieve
 ↓
LLM
 ↓
Answer
```

is often insufficient for complex research because the agent may need to:

* Break the goal into research questions
* Decide what to investigate next
* Use different research capabilities
* Evaluate whether evidence is sufficient
* Retry an unanswered question
* Stop when enough evidence has been collected
* Report questions that could not be answered

---

## Core Architecture

```text
User Goal
    ↓
Research Agent
    ↓
   PLAN
    ↓
  SELECT
    ↓
 RESEARCH
    ↓
 EVALUATE
    │
    ├── Enough evidence ──→ Complete Question
    │
    └── Insufficient ─────→ Research Again
                              │
                              ↓
                           SELECT
                              │
                              ↓
                           REPORT
```

The important design decision is that research is **question-driven** rather than one large retrieval loop.

---

# End-to-End Workflow

```text
1. User Request
       ↓
2. Create Initial State
       ↓
3. Plan Research
       ↓
4. Select First Research Question
       ↓
5. Research
       ↓
6. Evaluate Evidence
       │
       ├─ Sufficient
       │      ↓
       │   Complete Question
       │
       ├─ Insufficient + attempts remain
       │      ↓
       │   Research Again
       │
       └─ Attempts exhausted
              ↓
         Mark Question Exhausted

       ↓
7. Select Next Question
       │
       ├─ Question available → Research
       │
       └─ No question → Generate Report
```

### Important design property

One impossible question should **not trap the entire research process**.

For example:

```text
Question A → sufficient
Question B → sufficient
Question C → insufficient
Question C → retry
Question C → exhausted
Question D → sufficient
```

The system can still generate a useful report while explicitly reporting the limitation for Question C.

---

# State Design

There are two types of state:

### 1. Input / configuration state

This is supplied when the graph starts.

```text
goal
max_iterations
max_attempts_per_question
```

### 2. Runtime state

This is created and updated by graph nodes.

```text
required_questions
open_questions
current_question
evidence
question_attempts
exhausted_questions
research_count
research_complete
report
```

This distinction is important.

The caller should not have to know how the research agent internally manages questions, attempts, or evidence.

---

# LangGraph State

A simple implementation can use one state object:

```python
from typing import TypedDict


class Evidence(TypedDict):
    source: str
    content: str
    relevant: bool
    authoritative: bool
    supports_claim: bool


class ResearchState(TypedDict):
    # Input / configuration
    goal: str
    max_iterations: int
    max_attempts_per_question: int

    # Runtime state
    required_questions: list[str]
    open_questions: list[str]
    current_question: str | None

    evidence: dict[str, list[Evidence]]

    question_attempts: dict[str, int]
    exhausted_questions: list[str]

    research_count: int
    research_complete: bool

    report: str
```

The important point is:

> `TypedDict` defines the state schema. It does not initialize the values.

Therefore, the graph needs an initial state when invoked.

However, only the **input/configuration values are conceptually required from the caller**. The runtime fields should be initialized to safe defaults before execution.

---

# Initial State

The graph starts with:

```python
initial_state: ResearchState = {
    # Input / configuration
    "goal": "Assess ACME's security posture",
    "max_iterations": 10,
    "max_attempts_per_question": 2,

    # Runtime state
    "required_questions": [],
    "open_questions": [],
    "current_question": None,

    "evidence": {},

    "question_attempts": {},
    "exhausted_questions": [],

    "research_count": 0,
    "research_complete": False,

    "report": "",
}
```

Then:

```python
result = graph.invoke(initial_state)
```

The initial state is intentionally empty for research-specific fields because the **PLAN node owns their initialization**.

---

# First Iteration

The first execution is:

```text
Initial State
      ↓
     PLAN
      ↓
    SELECT
      ↓
   RESEARCH
      ↓
   EVALUATE
```

The important part is that the graph does **not** start with `SELECT` expecting questions to already exist.

`PLAN` creates them first.

---

# 1. Plan

The planning node determines what information must be known.

```python
def plan_research(state: ResearchState):
    questions = [
        "What assets are exposed?",
        "What technologies are being used?",
        "Are the technologies vulnerable?",
        "What is the security impact?",
    ]

    return {
        "required_questions": questions,
        "open_questions": questions.copy(),

        "question_attempts": {
            question: 0
            for question in questions
        },

        "exhausted_questions": [],
        "current_question": None,

        "evidence": {},
        "research_count": 0,
        "research_complete": False,
        "report": "",
    }
```

After `PLAN`, the state becomes conceptually:

```text
required_questions
 ├─ What assets are exposed?
 ├─ What technologies are being used?
 ├─ Are the technologies vulnerable?
 └─ What is the security impact?

open_questions
 ├─ Q1
 ├─ Q2
 ├─ Q3
 └─ Q4

question_attempts
 ├─ Q1 → 0
 ├─ Q2 → 0
 ├─ Q3 → 0
 └─ Q4 → 0
```

The planner therefore owns the initialization of the research plan.

---

# 2. Select

The selection node chooses the next question that still needs investigation.

```python
def select_question(state: ResearchState):
    for question in state["open_questions"]:

        attempts = state["question_attempts"].get(
            question,
            0,
        )

        if attempts < state["max_attempts_per_question"]:
            return {
                "current_question": question
            }

    return {
        "current_question": None
    }
```

For the first iteration:

```text
Q1
```

will be selected.

The state becomes:

```python
{
    "current_question": "What assets are exposed?"
}
```

---

# 3. Research

The research node executes the appropriate research capability.

```python
def discover_assets():
    return {
        "source": "asset_database",
        "content": "ACME exposes api.acme.com and app.acme.com",
        "relevant": True,
        "authoritative": True,
        "supports_claim": True,
    }


def identify_technologies():
    return {
        "source": "technology_inventory",
        "content": "api.acme.com uses nginx 1.24",
        "relevant": True,
        "authoritative": True,
        "supports_claim": True,
    }


def check_vulnerabilities():
    return {
        "source": "vulnerability_database",
        "content": "A known vulnerability affects nginx 1.24",
        "relevant": True,
        "authoritative": True,
        "supports_claim": True,
    }


def validate_risk():
    return {
        "source": "security_advisory",
        "content": "The vulnerability affects the exposed nginx service",
        "relevant": True,
        "authoritative": True,
        "supports_claim": True,
    }
```

Map questions to capabilities:

```python
RESEARCH_TOOLS = {
    "What assets are exposed?": discover_assets,
    "What technologies are being used?": identify_technologies,
    "Are the technologies vulnerable?": check_vulnerabilities,
    "What is the security impact?": validate_risk,
}
```

The research node:

```python
def research(state: ResearchState):

    question = state["current_question"]

    if question is None:
        return {}

    tool = RESEARCH_TOOLS.get(question)

    if tool is None:
        return {
            "research_count": state["research_count"] + 1
        }

    result = tool()

    existing_evidence = state["evidence"].get(
        question,
        [],
    )

    updated_evidence = {
        **state["evidence"],
        question: existing_evidence + [result],
    }

    updated_attempts = {
        **state["question_attempts"],
        question: state["question_attempts"].get(
            question,
            0,
        ) + 1,
    }

    return {
        "evidence": updated_evidence,
        "question_attempts": updated_attempts,
        "research_count": state["research_count"] + 1,
    }
```

The research node performs the action.

It does **not** decide whether the evidence is good enough.

---

# 4. Evaluate Evidence

The evaluator determines whether sufficient evidence exists.

```python
def evidence_is_sufficient(
    evidence: list[Evidence],
) -> bool:

    return any(
        item["relevant"]
        and item["authoritative"]
        and item["supports_claim"]
        for item in evidence
    )
```

Now evaluate all research questions:

```python
def evaluate_evidence(state: ResearchState):

    open_questions = []
    exhausted_questions = []

    for question in state["required_questions"]:

        evidence = state["evidence"].get(
            question,
            [],
        )

        if evidence_is_sufficient(evidence):
            continue

        attempts = state["question_attempts"].get(
            question,
            0,
        )

        if attempts >= state["max_attempts_per_question"]:
            exhausted_questions.append(question)
        else:
            open_questions.append(question)

    research_complete = len(open_questions) == 0

    return {
        "open_questions": open_questions,
        "exhausted_questions": exhausted_questions,
        "research_complete": research_complete,
    }
```

This creates an important separation:

```text
RESEARCH
Collect evidence

EVALUATE
Judge evidence
```

---

# 5. Route After Evaluation

The router determines what happens next.

```python
def route_after_evaluation(state: ResearchState):

    if state["research_complete"]:
        return "generate_report"

    if state["research_count"] >= state["max_iterations"]:
        return "generate_report"

    return "select_question"
```

There are two stopping conditions.

### Condition 1 — Research is complete

Every question is either:

```text
Sufficient
    OR
Exhausted
```

### Condition 2 — Global research limit reached

```text
research_count >= max_iterations
```

This protects against unexpected loops.

---

# 6. Generate Report

The report should include both findings and limitations.

```python
def generate_report(state: ResearchState):

    sections = []

    for question in state["required_questions"]:

        evidence = state["evidence"].get(
            question,
            [],
        )

        valid_evidence = [
            item
            for item in evidence
            if (
                item["relevant"]
                and item["authoritative"]
                and item["supports_claim"]
            )
        ]

        sections.append(f"## {question}")

        if not valid_evidence:
            sections.append(
                "No sufficient evidence was obtained."
            )
            continue

        for item in valid_evidence:
            sections.append(
                f"- {item['content']} "
                f"(Source: {item['source']})"
            )

    if state["exhausted_questions"]:

        sections.append("## Research Limitations")

        for question in state["exhausted_questions"]:
            sections.append(
                f"- Could not obtain sufficient evidence for: "
                f"{question}"
            )

    return {
        "report": "\n".join(sections)
    }
```

The agent should never silently turn missing evidence into a confident conclusion.

---

# Graph

```python
from langgraph.graph import StateGraph, START, END


builder = StateGraph(ResearchState)

builder.add_node("plan", plan_research)
builder.add_node("select_question", select_question)
builder.add_node("research", research)
builder.add_node("evaluate", evaluate_evidence)
builder.add_node("generate_report", generate_report)

builder.add_edge(START, "plan")
builder.add_edge("plan", "select_question")
builder.add_edge("select_question", "research")
builder.add_edge("research", "evaluate")

builder.add_conditional_edges(
    "evaluate",
    route_after_evaluation,
    {
        "select_question": "select_question",
        "generate_report": "generate_report",
    },
)

builder.add_edge("generate_report", END)

graph = builder.compile()
```

---

# Complete Execution

The complete lifecycle is:

```text
START
  ↓
PLAN
  │
  │ creates research questions
  ↓
SELECT
  │
  │ chooses next open question
  ↓
RESEARCH
  │
  │ executes research capability
  ↓
EVALUATE
  │
  ├── Evidence sufficient
  │        ↓
  │     Question complete
  │        ↓
  │     SELECT
  │
  ├── Evidence insufficient
  │   + attempts remain
  │        ↓
  │     SELECT same question
  │        ↓
  │     RESEARCH again
  │
  └── No open questions / limit reached
           ↓
        REPORT
           ↓
          END
```

---

# Example

Suppose the plan produces:

```text
Q1: What assets are exposed?
Q2: What technologies are being used?
Q3: Are the technologies vulnerable?
Q4: What is the security impact?
```

The execution may be:

```text
PLAN
 ↓
SELECT Q1
 ↓
RESEARCH Q1
 ↓
EVALUATE Q1
 ↓
SELECT Q2
 ↓
RESEARCH Q2
 ↓
EVALUATE Q2
 ↓
SELECT Q3
 ↓
RESEARCH Q3
 ↓
EVALUATE Q3
 ↓
SELECT Q4
 ↓
RESEARCH Q4
 ↓
EVALUATE Q4
 ↓
REPORT
```

If Q3 is insufficient:

```text
RESEARCH Q3
 ↓
EVALUATE
 ↓
Q3 insufficient
 ↓
attempts remain
 ↓
SELECT Q3
 ↓
RESEARCH Q3 again
```

If Q3 fails again:

```text
Q3
 ↓
attempt limit reached
 ↓
mark Q3 exhausted
 ↓
SELECT Q4
```

The research agent therefore continues rather than getting stuck on Q3.

---

# Production Concerns

## Source Quality

Research sources should have an explicit trust model.

For example:

```text
Authoritative
    ↓
Trusted internal source
    ↓
Secondary source
    ↓
Unverified source
```

The exact hierarchy depends on the domain.

## Evidence Provenance

Important evidence should retain:

```text
source
source_type
document_id / URL
retrieval_time
claim
evidence
```

This allows the final report to explain where a claim came from.

## Security

Research content is untrusted input.

Protect against:

* Prompt injection
* Malicious web pages
* Malicious documents
* Unauthorized retrieval
* Cross-tenant data access

## Runtime Limits

Use explicit limits:

```text
MAX_ITERATIONS
MAX_ATTEMPTS_PER_QUESTION
MAX_TOOL_CALLS
MAX_EXECUTION_TIME
MAX_TOKEN_BUDGET
MAX_COST
```

## Reliability

Use:

* Retry
* Timeout
* Fallback
* Checkpoint / Resume
* Idempotency where side effects exist

## Observability

Trace the complete research execution:

```text
Research Run
 ├─ Plan
 ├─ Question Selection
 ├─ Research / Tool
 ├─ Evidence Evaluation
 └─ Report Generation
```

## Evaluation

Evaluate both:

```text
Final Report
      +
Research Trajectory
```

A correct report does not necessarily mean the agent used an efficient, safe, or authorized research path.

---

# When to Use

Use a research agent when:

* The answer requires multiple sources
* The next research step depends on previous findings
* Evidence quality must be evaluated
* Missing information may require additional research
* The number of research steps cannot be known beforehand

Do not use a research agent when:

* One deterministic database query is sufficient
* A fixed RAG pipeline reliably answers the question
* The entire research path is known beforehand

---

# Trade-offs

| Benefit                        | Cost                  |
| ------------------------------ | --------------------- |
| Adaptive research              | Higher latency        |
| Multiple sources               | More infrastructure   |
| Evidence validation            | More model/tool calls |
| Handles missing information    | More complex state    |
| Supports long-horizon research | Higher cost           |
| Explicit limitations           | More evaluation logic |

---

# Mental Model

```text
PLAN
 ↓
What must be known?
 ↓
SELECT
 ↓
What should I investigate next?
 ↓
RESEARCH
 ↓
Collect evidence
 ↓
EVALUATE
 ↓
Is the evidence sufficient?
 │
 ├── No + attempts remain
 │       ↓
 │    SELECT
 │
 ├── No + attempts exhausted
 │       ↓
 │    Mark exhausted
 │       ↓
 │    SELECT next question
 │
 └── Yes / all questions complete
         ↓
       REPORT
```

> **Plan defines what needs to be known. Select decides what to investigate next. Research performs the action. Evaluate decides whether the evidence is sufficient. Report synthesizes the findings and limitations.**
