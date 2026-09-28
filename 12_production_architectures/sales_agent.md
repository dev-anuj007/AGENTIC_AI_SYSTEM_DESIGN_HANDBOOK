# Sales Agent

## Problem

A production sales platform may contain many capabilities:

* Account discovery
* Data normalization
* Feature extraction
* Signal detection
* Account scoring
* Company research
* Contact discovery
* Outreach generation
* CRM updates

Not all of these should be implemented as agents.

For example:

```text
Account data
    ↓
Normalize
    ↓
Extract features
    ↓
Detect signals
    ↓
Calculate score
```

If these steps have known rules and known execution paths, they are better implemented as **deterministic services or workflows**.

An agent becomes useful when the system needs to dynamically decide:

* What should I investigate?
* Which source should I use?
* Which tool should I call?
* What information is still missing?
* Is the evidence sufficient?
* What should I investigate next?

Therefore, a production sales architecture is usually **hybrid** rather than "everything is an agent."

---

# Core Architecture

Consider a cybersecurity sales intelligence platform.

The architecture can be divided into three major parts:

```text
                 SALES INTELLIGENCE PLATFORM
                           │
             ┌─────────────┴─────────────┐
             │                           │
      DETERMINISTIC PIPELINE       AGENTIC RESEARCH
             │                           │
     Account Discovery                  Goal
             ↓                           ↓
     Data Normalization             Plan Research
             ↓                           ↓
     Feature Extraction          Select Question/Tool
             ↓                           ↓
     Signal Detection                 Research
             ↓                           ↓
     Initial Scoring                 Evaluate
             │                       ┌───┴───┐
             │                       │       │
             │                  Sufficient  Missing
             │                       │       │
             │                       │    Research Again
             │                       │
             └───────────────┬───────┘
                             ↓
                    Opportunity Context
                             ↓
                      OUTREACH AGENT
                             ↓
                     Policy / Approval
                             ↓
                       External Action
```

The important boundary is:

```text
Known execution path
        ↓
    Workflow

Unknown execution path
        ↓
      Agent
```

---

# Example

Suppose the platform discovers:

```text
Company: ACME

Detected signals:

- nginx detected
- nginx version 1.24
- known vulnerability associated with version 1.24
- public API endpoint exposed
```

The deterministic pipeline can produce these facts.

But the sales system may still need to answer:

> Is this actually a meaningful cybersecurity opportunity?

That question may require additional investigation.

The research agent could decide:

```text
1. Verify whether the affected service is externally exposed.
2. Determine whether the vulnerable version is still active.
3. Determine whether the affected asset appears business-critical.
4. Look for additional security context.
5. Stop when sufficient evidence exists.
```

The agent determines the **research path**.

The underlying tools remain controlled application capabilities.

---

# Where the Agent Actually Fits

A useful architecture is:

```text
                 ACCOUNT DATA
                      ↓
              SIGNAL PIPELINE
                      ↓
              INITIAL SIGNALS
                      ↓
               RESEARCH AGENT
                      ↓
        ┌─────────────┴─────────────┐
        │                           │
  Research sufficient        Research insufficient
        │                           │
        ↓                           ↓
 Opportunity Context          More Research
        │                           │
        └─────────────┬─────────────┘
                      ↓
               Outreach Agent
```

The research agent should **not** own the entire sales platform.

It owns the problem:

> Gather enough evidence to answer a research question.

---

# End-to-End Execution Example

Consider this real execution:

```text
Goal:
Identify whether ACME is a meaningful cybersecurity
sales opportunity based on detected infrastructure signals.
```

The execution begins **before the agent**.

## Step 1 — Account Discovery

The platform queries its account source.

```text
Account Discovery Service
        ↓
ACME
```

The result:

```json
{
  "account_id": "account-123",
  "name": "ACME",
  "industry": "SaaS",
  "employees": 850
}
```

No agent is required here.

The query is deterministic.

---

## Step 2 — Data Processing

The platform processes available infrastructure data.

```text
Raw Infrastructure Data
        ↓
Normalization
        ↓
Feature Extraction
        ↓
Signal Detection
```

Suppose it discovers:

```text
api.acme.com
    ↓
nginx
    ↓
version 1.24
```

The signal engine checks its vulnerability data:

```text
nginx 1.24
    ↓
Known vulnerability
```

The platform creates:

```json
{
  "type": "known_vulnerability",
  "product": "nginx",
  "version": "1.24",
  "asset": "api.acme.com",
  "source": "vulnerability_database"
}
```

Still no agent is required.

---

## Step 3 — Initial Opportunity Detection

The deterministic scoring pipeline sees:

```text
ICP match              ✓
Technology signal      ✓
Known vulnerability    ✓
Public asset           ✓
```

It creates an initial research task:

```text
Research Goal:

Determine whether the detected vulnerability
represents a meaningful cybersecurity opportunity.
```

Now the **agentic portion begins**.

---

# Step 4 — Create Agent State

The platform gives the research agent:

```text
Account:
ACME

Known facts:
- SaaS company
- 850 employees
- api.acme.com exists
- nginx 1.24 detected
- vulnerability associated with nginx 1.24

Goal:
Determine whether the vulnerability represents
a meaningful opportunity.
```

Initial state:

```python
{
    "goal": (
        "Determine whether the detected vulnerability "
        "represents a meaningful opportunity."
    ),

    "account_id": "account-123",

    "existing_signals": [
        {
            "type": "known_vulnerability",
            "product": "nginx",
            "version": "1.24",
            "asset": "api.acme.com",
        }
    ],

    "research_questions": [],
    "open_questions": [],
    "current_question": None,

    "evidence": [],

    "tool_calls": 0,
    "iterations": 0,

    "research_complete": False,
    "conclusion": "",
}
```

---

# Step 5 — Agent Plans Research

The agent determines what must be established.

```text
PLAN

Question 1:
Is nginx 1.24 still active?

Question 2:
Is api.acme.com externally exposed?

Question 3:
Is the affected service associated with production?

Question 4:
Does the vulnerability actually apply to
the deployed configuration?
```

Notice that these questions were created from the current context.

They were not simply another fixed sales pipeline step.

---

# Step 6 — Agent Selects First Action

The agent decides:

```text
Current question:

Is nginx 1.24 still active?
```

It selects:

```text
get_technology_version()
```

The tool gateway checks authorization.

```text
Agent
 ↓
Tool Request
 ↓
Authorization
 ↓
get_technology_version()
```

Result:

```json
{
  "asset": "api.acme.com",
  "product": "nginx",
  "version": "1.24",
  "observed_at": "2026-09-25"
}
```

The agent records the evidence.

---

# Step 7 — Agent Evaluates Evidence

The agent now knows:

```text
nginx 1.24 is currently active
```

But this does not answer the complete research goal.

The agent determines that another question remains:

```text
Is the affected service externally exposed?
```

It selects:

```text
get_asset_details()
```

Result:

```json
{
  "asset": "api.acme.com",
  "internet_exposed": true,
  "environment": "production"
}
```

Now the evidence becomes:

```text
nginx 1.24
     +
currently active
     +
internet exposed
     +
production
```

---

# Step 8 — Agent Changes Research Direction

The agent now asks:

> Is the vulnerability actually applicable to this deployment?

It calls:

```text
check_vulnerability()
```

Result:

```json
{
  "product": "nginx",
  "version": "1.24",
  "affected": true,
  "severity": "high",
  "source": "security_advisory"
}
```

The agent now has strong technical evidence.

But it may still need business context.

It decides to investigate:

```text
Is this production API associated with
a business-critical service?
```

It calls:

```text
get_business_context()
```

Result:

```json
{
  "asset": "api.acme.com",
  "service": "customer_api",
  "environment": "production",
  "business_criticality": "high"
}
```

---

# Step 9 — Agent Determines Evidence Is Sufficient

The agent now has:

```text
✓ Vulnerable technology confirmed
✓ Vulnerable version confirmed
✓ Version currently active
✓ Publicly exposed
✓ Production environment
✓ High business criticality
✓ Vulnerability applies to deployed version
```

The agent evaluates the research goal:

```text
Is this a meaningful cybersecurity opportunity?

YES — sufficient evidence.
```

The research loop terminates.

This is the important agentic behavior:

```text
Initial evidence
      ↓
Question
      ↓
Tool
      ↓
Observation
      ↓
New question
      ↓
Tool
      ↓
Observation
      ↓
Evaluate
      ↓
New question
      ↓
Tool
      ↓
Sufficient evidence
```

The exact sequence was determined by what the agent discovered.

---

# Step 10 — Agent Returns Structured Result

The agent returns:

```json
{
  "account_id": "account-123",

  "conclusion": "meaningful_opportunity",

  "confidence": 0.94,

  "evidence": [
    {
      "fact": "nginx 1.24 is currently active",
      "source": "technology_inventory"
    },
    {
      "fact": "api.acme.com is publicly exposed",
      "source": "asset_inventory"
    },
    {
      "fact": "The vulnerability affects nginx 1.24",
      "source": "security_advisory"
    },
    {
      "fact": "The asset belongs to a production,
               high-criticality service",
      "source": "business_context"
    }
  ],

  "limitations": []
}
```

The agent has finished its responsibility.

It produced **validated research context**.

---

# Step 11 — Deterministic Opportunity Scoring

The platform receives the agent result.

The scoring service now calculates the opportunity score.

For example:

```text
ICP Fit                  30
Validated Vulnerability  30
Public Exposure          20
Production Criticality   20
                         ──
                         100
```

The scoring service returns:

```json
{
  "account_id": "account-123",
  "opportunity_score": 100,
  "qualification": "qualified"
}
```

The important point is that the score does **not** have to be generated by the agent.

The agent supplied validated evidence.

The scoring system applies business rules.

---

# Step 12 — Outreach Agent

The qualified opportunity is passed to the outreach agent.

Input:

```text
Account:
ACME

Validated facts:
- nginx 1.24 active
- public API
- vulnerability applies
- production service
- high business criticality

Product:
Cybersecurity platform
```

The outreach agent generates a draft.

Example:

```text
Subject:
Security consideration for api.acme.com

Body:

We identified that api.acme.com is currently running
nginx 1.24, a version associated with a known security
advisory.

Because the service is publicly exposed and appears to
support a production workload, it may be worth reviewing
the affected deployment and remediation options.
```

The draft is grounded in the research evidence.

---

# Step 13 — Policy and Approval

The system does not automatically send the message merely because the agent generated it.

Instead:

```text
Outreach Draft
      ↓
Policy Check
      ↓
Approval Required?
      │
      ├── No → Send
      │
      └── Yes
           ↓
        Human Approval
           ↓
          Send
```

If approval is required:

```text
Sales Representative
        ↓
Approve
        ↓
Email Provider
```

The email service performs the actual side effect.

---

# Complete Execution

The entire system therefore executes as:

```text
┌─────────────────────────────────────────────────────────────┐
│                  DETERMINISTIC PLATFORM                    │
│                                                             │
│ Account Discovery                                           │
│       ↓                                                     │
│ Data Normalization                                          │
│       ↓                                                     │
│ Feature Extraction                                          │
│       ↓                                                     │
│ Signal Detection                                            │
│       ↓                                                     │
│ Initial Opportunity Context                                 │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                     RESEARCH AGENT                         │
│                                                             │
│ Goal                                                        │
│  ↓                                                          │
│ Plan                                                        │
│  ↓                                                          │
│ Select Question                                             │
│  ↓                                                          │
│ Select Tool                                                 │
│  ↓                                                          │
│ Research                                                    │
│  ↓                                                          │
│ Evaluate Evidence                                           │
│  │                                                          │
│  ├── Insufficient → Select Next Question → Research         │
│  │                                                          │
│  └── Sufficient → Research Complete                         │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                  DETERMINISTIC SCORING                     │
│                                                             │
│ Validated Evidence                                          │
│       ↓                                                     │
│ Business Rules                                              │
│       ↓                                                     │
│ Opportunity Score                                           │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                     OUTREACH AGENT                         │
│                                                             │
│ Account Context                                             │
│       ↓                                                     │
│ Validated Evidence                                          │
│       ↓                                                     │
│ Generate Personalized Draft                                 │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                POLICY / HUMAN APPROVAL                     │
│                                                             │
│ Authorization                                               │
│       ↓                                                     │
│ Approval                                                    │
│       ↓                                                     │
│ External Action                                             │
└─────────────────────────────────────────────────────────────┘
```

---

# Why This Is Agentic

The important part is not that an LLM appears in the architecture.

The agentic behavior is visible here:

```text
Agent receives:
    Goal + Current Evidence + Available Tools

             ↓

Agent decides:
    What information is missing?

             ↓

Agent chooses:
    Which question should be investigated?

             ↓

Agent chooses:
    Which tool can answer it?

             ↓

Tool executes

             ↓

Agent observes result

             ↓

Agent decides:
    Is evidence sufficient?

             ↓

        ┌────┴────┐
        ↓         ↓
       YES        NO
        ↓         ↓
     Finish    Investigate Again
```

That dynamic decision loop is what differentiates the research agent from a fixed workflow.

---

# State Design

The state should represent the **research task**, not the entire sales platform.

```python
from typing import TypedDict


class Evidence(TypedDict):
    source: str
    content: str
    relevant: bool
    authoritative: bool


class ResearchState(TypedDict):
    # Input / configuration
    goal: str
    account_id: str

    max_iterations: int
    max_tool_calls: int

    # Context supplied by the platform
    existing_signals: list[dict]

    # Runtime state
    research_questions: list[str]
    open_questions: list[str]

    current_question: str | None

    evidence: list[Evidence]

    tool_calls: int
    iterations: int

    research_complete: bool
    conclusion: str
```

Initial state:

```python
initial_state: ResearchState = {
    "goal": (
        "Determine whether the detected vulnerability "
        "represents a meaningful opportunity."
    ),

    "account_id": "account-123",

    "max_iterations": 6,
    "max_tool_calls": 10,

    "existing_signals": [
        {
            "type": "known_vulnerability",
            "product": "nginx",
            "version": "1.24",
            "asset": "api.acme.com",
        }
    ],

    "research_questions": [],
    "open_questions": [],

    "current_question": None,

    "evidence": [],

    "tool_calls": 0,
    "iterations": 0,

    "research_complete": False,
    "conclusion": "",
}
```

The important distinction is that the **sales platform owns the account data and initial signals**.

The research agent owns its research state.

---

# Agent Toolset

The research agent might receive capabilities such as:

```text
get_asset_details
get_technology_version
check_vulnerability
search_company_information
search_security_advisories
get_business_context
```

These are tools.

The agent decides when they are useful.

The tool gateway remains responsible for:

```text
Authentication
Authorization
Input validation
Rate limiting
Timeout
Audit
Tenant isolation
```

---

# Opportunity Scoring

Scoring does not necessarily belong inside the agent.

If the organization's scoring model is deterministic:

```text
ICP Fit
+
Validated Signals
+
Signal Recency
+
Business Relevance
=
Opportunity Score
```

then implement it as a scoring service.

The agent provides **validated evidence** to the scoring service.

It does not need to replace the scoring system.

---

# Outreach Agent

Outreach is a separate concern.

The outreach agent may receive:

```text
Account
Validated signals
Opportunity context
Product information
Communication policy
```

It can then produce:

```text
Draft
 ↓
Evidence Check
 ↓
Policy Check
 ↓
Human Approval
 ↓
Send
```

The outreach agent should not invent facts.

---

# Production Architecture

A realistic production system therefore looks like:

```text
                         Account Sources
                               ↓
                    Deterministic Pipeline
                               ↓
                     Features / Signals
                               ↓
                       Research Agent
                               ↓
                     Validated Evidence
                               ↓
                       Scoring Service
                               ↓
                        Opportunity
                               ↓
                       Outreach Agent
                               ↓
                       Policy / Approval
                               ↓
                      CRM / Email / etc.
```

Each component has a different responsibility.

| Component        | Responsibility                              |
| ---------------- | ------------------------------------------- |
| Data pipeline    | Normalize and process data                  |
| Signal engine    | Detect known signals                        |
| Research agent   | Dynamically investigate missing information |
| Scoring service  | Apply business scoring rules                |
| Outreach agent   | Generate grounded communication             |
| Policy layer     | Decide whether action is permitted          |
| External service | Perform the actual side effect              |

---

# Reliability

The research agent should have explicit limits:

```text
MAX_ITERATIONS
MAX_TOOL_CALLS
MAX_EXECUTION_TIME
MAX_TOKEN_BUDGET
MAX_COST
```

For external tools:

```text
Timeout
Retry
Fallback
Circuit breaker
```

For long-running research:

```text
Checkpoint
Resume
```

For external side effects:

```text
Idempotency
Authorization
Audit
```

---

# Security

The research agent may access sensitive customer information.

Important controls include:

```text
Authentication
Authorization
Tenant Isolation
Tool Permissions
Prompt Injection Protection
Data Filtering
Audit Logging
```

External research sources must be treated as untrusted data.

A webpage can provide evidence, but it cannot authorize an action.

---

# Observability

A useful trace should show:

```text
Sales Research Agent
 ├── Research Question
 ├── Tool Selection
 ├── Tool Call
 ├── Tool Result
 ├── Evidence Evaluation
 ├── Next Question
 └── Final Conclusion
```

Important metrics:

```text
research_iterations
tool_calls
research_latency
token_usage
cost
evidence_count
research_success_rate
```

---

# When to Use

Use a sales agent when:

* Research requires multiple sources
* The next research action depends on previous findings
* The required evidence is not known in advance
* Different accounts require different investigation paths
* Evidence must be evaluated before reaching a conclusion

Use a deterministic workflow instead when:

* The steps are known in advance
* Rules completely define the decision
* A database query is sufficient
* A fixed ETL/data-processing pipeline solves the problem

---

# Trade-offs

| Agentic Research              | Deterministic Workflow |
| ----------------------------- | ---------------------- |
| Adaptive                      | Predictable            |
| Handles unknown paths         | Easier to test         |
| Can investigate dynamically   | Lower cost             |
| More flexible                 | Easier to audit        |
| Higher latency                | Lower latency          |
| Higher operational complexity | Simpler infrastructure |

The goal is not to maximize the amount of agentic behavior.

The goal is to put agentic behavior **only where dynamic decision-making provides value**.

---

# Mental Model

```text
                KNOWN
                  │
                  ↓
            Deterministic
              Pipeline
                  │
                  ↓
        "What facts/signals exist?"
                  │
                  ↓
             UNKNOWN
                  │
                  ↓
          Research Agent
                  │
       ┌──────────┴──────────┐
       ↓                     ↓
   What do I know?      What is missing?
       │                     │
       └──────────┬──────────┘
                  ↓
             Research
                  ↓
             Evaluate
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
   Sufficient            Missing
        ↓                   ↓
    Complete          Research Again
        │
        ↓
  Validated Evidence
        ↓
 Deterministic Scoring
        ↓
  Outreach Agent
        ↓
 Policy / Approval
        ↓
     External Action
```

> **A sales agent should not replace the sales intelligence pipeline. It should handle the parts of sales intelligence where the path to the answer is unknown and requires adaptive research.**
