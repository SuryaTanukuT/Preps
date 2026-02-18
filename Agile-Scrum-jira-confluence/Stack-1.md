https://www.interviewbit.com/agile-interview-questions/
https://www.igmguru.com/blog/jira-interview-questions
https://www.geeksforgeeks.org/interview-experiences/agile-software-development-interview-questions/
https://beknazarsuranchiyev.medium.com/sdlc-agile-interview-questions-for-sdet-eaf8051bf5b1

https://www.gsdcouncil.org/blogs/top-agile-testing-interview-questions-to-ace-your-next-job

---

# Agile

## What Agile means (real definition)

Agile is a **delivery mindset**: build in small increments, validate early, adapt continuously.

### Core principles you should mention

* **Iterative delivery** (small slices)
* **Customer feedback loop**
* **Embrace change**
* **Working software > documentation** (but not “no docs”)
* **Cross-functional teams**
* **Continuous improvement** (retros)

### Agile in practice (what interviewers want)

* “We deliver in **sprints**, keep **visible backlog**, measure **cycle time/lead time**, and continuously refine scope with stakeholders.”

---

# Scrum (Agile framework)

## Scrum in one line

Scrum is a framework to deliver value in **time-boxed sprints** using **roles + events + artifacts**.

---

## Roles (who does what)

### **Product Owner (PO)**

* Owns **product backlog**
* Defines **priority + value**
* Clarifies requirements
* Accepts completed work

### **Scrum Master**

* Facilitates Scrum events
* Removes impediments
* Coaches team on Scrum
* Protects team from churn

### **Development Team**

* Builds increment
* Owns estimates + delivery
* Ensures quality

**Senior angle:** As a lead, you act as “execution glue”: clarify scope, reduce ambiguity, unblock, ensure quality gates.

---

## Events (meetings) and purpose

### 1) **Sprint Planning**

Goal: decide **what** to deliver + **how**
Outputs:

* Sprint Goal
* Selected backlog items
* Initial task breakdown

### 2) **Daily Standup (15 mins)**

Goal: sync + unblock
Format (modern):

* What I did / what I’ll do / blockers
* (or) “progress towards sprint goal + blockers”

### 3) **Backlog Refinement (Grooming)**

Goal: keep backlog “ready”

* Clarify acceptance criteria
* Break down epics → stories
* Estimate
* Identify dependencies

### 4) **Sprint Review**

Goal: demo to stakeholders + collect feedback
Output: reprioritized backlog

### 5) **Sprint Retrospective**

Goal: improve process

* What went well
* What didn’t
* Action items (owned + measurable)

---

## Artifacts (deliverables)

### **Product Backlog**

All work, prioritized.

### **Sprint Backlog**

Subset selected for sprint + plan.

### **Increment**

Potentially shippable product output.

---

## Definitions you should always mention

### Definition of Ready (DoR)

A story is “ready” when:

* clear description
* acceptance criteria
* dependencies known
* testable
* sized reasonably

### Definition of Done (DoD)

Done means:

* code complete
* tests passed (unit/integration)
* code review done
* security checks done
* deployed to env (as per org)
* docs updated if needed

---

# Jira (execution + tracking)

## What Jira is

Jira is a tool to manage:

* backlog
* sprints
* tasks/bugs
* workflow states
* reporting

---

## Core Jira objects

### Issue types

* **Epic** → big feature
* **Story** → user-focused value chunk
* **Task** → technical work (refactor, infra)
* **Bug** → defect
* **Spike** → research / POC

### Hierarchy

Epic → Stories/Tasks → Subtasks

---

## Workflow (typical)

* To Do → In Progress → Code Review → QA → Done
  (Or your org’s variant)

**Senior note:** Keep workflow “thin” to avoid slow moving tickets.

---

## Estimation methods in Jira

### Story points (most common)

* Relative estimation (Fibonacci: 1,2,3,5,8…)
* Based on complexity + risk + unknowns

### Time-based (less ideal)

Hours/days for tasks (common in ops teams)

---

## Jira boards

### Scrum board

* Sprint-based
* backlog + sprint planning
* burndown chart

### Kanban board

* Continuous flow
* WIP limits
* cycle time focus

---

## Important Jira metrics (interview winners)

* **Velocity**: points completed per sprint (trend, not target)
* **Burndown/Burnup**
* **Cycle time**: start → done
* **Lead time**: request created → done
* **WIP**: work in progress (lower is better)

**Senior stance:**

> “I focus on cycle time + WIP limits, not just velocity.”

---

# Confluence (documentation + knowledge base)

## What Confluence is

Confluence is a central place to store:

* requirements
* design docs
* runbooks
* meeting notes
* postmortems
* architecture decisions

---

## Documents that matter in real teams

* **PRD / Feature spec**
* **HLD/LLD**
* **API contract**
* **Runbooks** (on-call / ops)
* **Release notes**
* **RCA / postmortem**
* **ADRs** (Architecture Decision Records)

---

## Best practice: Jira ↔ Confluence linking

* Jira ticket links to:

  * PRD/spec page
  * design doc
  * test plan / rollout plan
* Confluence page links back to:

  * Epic / sprint board
  * related tickets
  * decision log

This prevents “tribal knowledge”.

---

# How Agile + Scrum + Jira + Confluence fit together (most important)

### Example flow

1. PO writes **Epic** in Jira
2. Confluence has **PRD + requirements**
3. Team refines stories (DoR)
4. Sprint planning → pick stories into sprint
5. Dev tracks work in Jira workflow
6. Sprint review demo + update docs
7. Retro produces action items
8. Confluence stores decisions + runbooks + postmortems

---

# Senior interview answers (copy/paste)

### “What’s your Agile process?”

> “We follow Scrum with 2-week sprints: planning, daily standups, refinement, review and retro. Jira manages backlog/sprints and Confluence holds PRDs, design docs, ADRs and runbooks. We measure cycle time/WIP and keep DoR/DoD strict to maintain predictability.”

### “How do you handle requirement changes mid-sprint?”

> “We protect the sprint goal. Small changes can be swapped with equal effort; major scope changes go back to backlog and are prioritized for the next sprint unless it’s production-critical.”

### “How do you ensure quality in Agile delivery?”

> “Definition of Done includes code review, automated tests, security checks, and deployment readiness. We keep stories small, shift left on testing, and use CI/CD gates.”

---


---

# 1) Sprint Planning — what you do + what you say

## 🎯 Your objective (Senior Lead lens)

* Lock a **Sprint Goal** (business outcome, not a list of tickets)
* Ensure stories meet **DoR**
* Identify **dependencies / risks / compliance**
* Create a realistic plan with capacity + buffers

## ✅ What you say (interview-ready)

> “In sprint planning, we agree on a sprint goal, pull top-priority stories that meet DoR, confirm acceptance criteria, and identify dependencies early—especially around data, integration, and security. For BFSI flows, we explicitly review audit logging, idempotency, and rollback strategy as part of the plan.”

## 🧩 Planning checklist (quick)

* Do we have **clear acceptance criteria**?
* Any **external dependency** (API, DB migration, another team)?
* Any **risk** (payment correctness, money movement, PII)?
* Any **non-functional** constraints (latency, rate-limits, retries)?
* Is testing strategy clear (**unit + integration + contract**)?
* Rollout strategy: **feature flag / canary / rollback**

## 🗣️ Sprint goal examples (BFSI)

* “Enable idempotent payment capture + audit trail”
* “Reduce settlement failure rate by improving retry + DLQ”
* “Add maker-checker approval workflow for high-risk ops”

---

# 2) Daily Standup — short, crisp, unblocker-driven

## 🎯 Your objective

* Align on progress toward sprint goal
* Surface blockers fast
* Prevent hidden “stuck” work

## ✅ Best modern format (stronger than 3 questions)

**1) Progress toward sprint goal**
**2) What’s blocked / risky**
**3) Next 24 hours + who needs help**

## 🗣️ What you say (as Lead)

> “I focus standup on the sprint goal. I look for blocked items, risk hotspots, and integration points. If something is waiting on another team or environment, I pull it out immediately and resolve it outside standup.”

## 🔥 Senior signals during standup

* “This is a dependency, let’s sync after standup with X.”
* “This could impact prod stability; let’s add a rollback plan.”
* “This has NFR risk; let’s add a quick load test/check.”
* “We’re exceeding WIP; let’s finish before starting new.”

---

# 3) Backlog Refinement — where seniors win

## 🎯 Your objective

* Turn fuzzy ideas into “ready” work
* Reduce sprint churn
* Prevent scope surprises

## ✅ What you say (interview-ready)

> “Refinement is where we de-risk delivery. We break epics into thin vertical slices, confirm acceptance criteria, define edge cases, and align on integration contracts. In BFSI, we always refine around failure modes: retries, duplicates, partial failures, auditability, and reconciliation.”

## 🧱 Story slicing patterns (microservices)

* Slice by **workflow step** (validate → authorize → capture)
* Slice by **API contract** (contract first, then implementation)
* Slice by **risk** (idempotency + audit first, features later)

## Refinement checklist (DoR)

* User story + business context
* Acceptance criteria (happy path + failure cases)
* Data impact (schema/migration)
* Observability (logs/metrics/traces)
* Security (PII masking, access control)
* Test approach
* Estimated + small enough

---

# 4) Sprint Review — demo like a product engineer, not a coder

## 🎯 Your objective

* Demonstrate working increment
* Capture stakeholder feedback
* Decide next priorities

## ✅ What you say

> “In sprint review, we demo end-to-end behavior, not just endpoints. We show success + failure flows, monitoring/metrics for critical transactions, and confirm acceptance criteria with the PO. Feedback is converted into backlog updates immediately.”

## 🧪 BFSI demo style (gold)

Demo these explicitly:

* Happy flow
* Duplicate request behavior (idempotency)
* Error paths (timeouts, downstream failures)
* Audit log entry created
* Metrics dashboard signal (latency, error rate)
* Feature flag toggling + rollback readiness

---

# 5) Retrospective — action-driven, not a complaint session

## 🎯 Your objective

* Improve flow
* Reduce cycle time
* Raise engineering maturity

## ✅ What you say (interview-ready)

> “Retro is about measurable improvement. We choose 1–3 actions max, assign owners, define success metrics, and review them next retro. Typical actions: reduce WIP, tighten DoD, improve CI reliability, add contract tests, or strengthen runbooks.”

## Retro format that sounds senior

* Start / Stop / Continue
* 4Ls (Liked, Learned, Lacked, Longed for)
* Root cause mini-RCA for recurring issues

## Examples of strong retro actions

* “Add contract tests for payment gateway integration”
* “Introduce idempotency keys for all money-moving endpoints”
* “Add alerting on settlement queue lag + DLQ spikes”
* “Reduce PR review SLA to <24 hours”
* “Define DoD requiring tracing + dashboards for critical APIs”

---

# 6) Jira — what a Senior Lead actually manages

## 🎯 Your objective

* Keep flow visible
* Prevent ticket rot
* Maintain predictable delivery

## 🗣️ What you say

> “I keep Jira clean and outcome-driven: epics mapped to goals, stories small and testable, clear acceptance criteria, and minimal workflow states. I use cycle time and WIP as primary signals, not velocity gaming.”

## Strong Jira habits

* “One story = one deployable slice”
* Avoid huge stories; prefer 2–3 day slices
* Track dependencies explicitly
* Bugs: severity + RCA link
* Always link Confluence spec/design to epic/story

---

# 7) Confluence — what you document (BFSI style)

## What you say (interview-ready)

> “Confluence is our single source of truth. PRD, design docs, ADRs, runbooks, and postmortems live there. Every epic links to a spec/design page, and production changes link to rollout + rollback steps.”

## Minimum docs that impress

* PRD / feature spec
* HLD/LLD (even lightweight)
* ADRs (why we chose Kafka vs RabbitMQ etc.)
* Runbook (alerts, dashboards, common fixes)
* Postmortem template (RCA + actions)

---

# 8) The “BFSI-grade” add-ons (this is your differentiator)

When asked “How do you ensure reliability?” say:

* **Idempotency** for money movement
* **Outbox / Saga** patterns for consistency
* **Audit trails** (immutable logs)
* **Reconciliation jobs**
* **Retries + DLQ** for async flows
* **Feature flags + canary + rollback**
* **Observability**: tracing + metrics + alerts

---

# 9) 30-second Scrum + tool pitch (use this verbatim)

> “We run 2-week Scrum sprints. Jira handles backlog, sprint planning, and flow tracking; Confluence holds PRDs, designs, ADRs, and runbooks. My focus as a lead is sprint goal clarity, de-risking work in refinement, enforcing DoR/DoD, and ensuring BFSI-grade reliability—idempotency, audit trails, observability, and safe rollout/rollback.”

---
## Most important Scrum ceremonies (and why they matter)

### 1) **Sprint Planning** (highest leverage)

**Why it’s critical:** This is where you lock the **Sprint Goal**, align on “what good looks like,” and de-risk delivery (dependencies, capacity, testing, rollout).
**Outputs that matter:**

* Clear **Sprint Goal**
* Stories that meet **Definition of Ready**
* A realistic plan + shared understanding of scope, risks, and ownership

---

### 2) **Daily Scrum** (flow-control, not status reporting)

**Why it’s critical:** Keeps the team moving toward the Sprint Goal and exposes blockers early.
**Signals it’s healthy:**

* Short (10–15 mins), goal-focused
* Blockers are surfaced fast and resolved outside the meeting
* WIP stays controlled (people finish work before starting new work)

---

### 3) **Sprint Review** (value validation)

**Why it’s critical:** This is where you prove you delivered **working software** and get **stakeholder feedback** early.
**Healthy output:**

* Demo of potentially shippable increment
* Feedback translated into backlog changes
* Stakeholders aligned on next priorities

---

### 4) **Retrospective** (continuous improvement engine)

**Why it’s critical:** Without retro, teams plateau and repeat the same pain.
**Healthy output:**

* 1–3 concrete improvements with owners + due dates
* Follow-up on last retro actions
* Focus on system/process improvements, not blame

---

### 5) **Backlog Refinement** (often the hidden “make or break”)

**Why it’s critical:** Prevents sprint churn and confusion. Refinement is where you convert “ideas” into “ready work.”
**Healthy output:**
## Most important Scrum ceremonies (and why they matter)

### 1) **Sprint Planning** (highest leverage)

**Why it’s critical:** This is where you lock the **Sprint Goal**, align on “what good looks like,” and de-risk delivery (dependencies, capacity, testing, rollout).
**Outputs that matter:**

* Clear **Sprint Goal**
* Stories that meet **Definition of Ready**
* A realistic plan + shared understanding of scope, risks, and ownership

---

### 2) **Daily Scrum** (flow-control, not status reporting)

**Why it’s critical:** Keeps the team moving toward the Sprint Goal and exposes blockers early.
**Signals it’s healthy:**

* Short (10–15 mins), goal-focused
* Blockers are surfaced fast and resolved outside the meeting
* WIP stays controlled (people finish work before starting new work)

---

### 3) **Sprint Review** (value validation)

**Why it’s critical:** This is where you prove you delivered **working software** and get **stakeholder feedback** early.
**Healthy output:**

* Demo of potentially shippable increment
* Feedback translated into backlog changes
* Stakeholders aligned on next priorities

---

### 4) **Retrospective** (continuous improvement engine)

**Why it’s critical:** Without retro, teams plateau and repeat the same pain.
**Healthy output:**

* 1–3 concrete improvements with owners + due dates
* Follow-up on last retro actions
* Focus on system/process improvements, not blame

---

### 5) **Backlog Refinement** (often the hidden “make or break”)

**Why it’s critical:** Prevents sprint churn and confusion. Refinement is where you convert “ideas” into “ready work.”
**Healthy output:**

* Clear acceptance criteria + edge cases
* Right-sized stories
* Dependencies discovered early
* Reduced mid-sprint surprises

> If I must pick **top 3 by impact**: **Refinement + Planning + Retro** (because they prevent chaos).
> If I must pick **top 3 by Scrum purity**: **Planning + Daily + Review + Retro** (core Scrum events).

---

# How to gauge a Scrum team’s success (Senior Lead way)

## A) Value outcomes (most important)

**Ask:** Are we delivering meaningful outcomes, not just closing tickets?
**Measures:**

* Stakeholder satisfaction / adoption
* Feature usage metrics, conversion, revenue impact (if applicable)
* Reduction in incidents, support tickets, or operational load
* Clear Sprint Goals achieved consistently

---

## B) Predictability & reliability

**Ask:** Can we make commitments and meet them sustainably?
**Signals:**

* Sprint Goals met most sprints (even if scope changes)
* Stable throughput over time (not extreme spikes)
* Low “carry-over” / spillover stories
* Work gets to “Done” (including testing + release), not “almost done”

---

## C) Flow efficiency (how fast work moves)

**Ask:** Is work flowing smoothly from “ready” → “done”?
**Metrics that actually help:**

* **Cycle time** (start → done)
* **Lead time** (request → done)
* **WIP** (too high = congestion)
* Blocker age, time-to-unblock
* % work stuck in review/QA environments

> Healthy teams optimize **cycle time + WIP**, not velocity.

---

## D) Quality & engineering health

**Ask:** Are we shipping without breaking production?
**Signals:**

* Low escaped defects
* Incident rate trending down
* Good test signal (unit/integration/contract where relevant)
* Clean rollback / feature flags / safe releases
* Tech debt is managed (not ignored)

---

## E) Team maturity & continuous improvement

**Ask:** Are retros actually changing things?
**Signals:**

* Retro actions are implemented and reviewed
* Less repeated pain (fewer recurring blockers/incidents)
* Collaboration improves (less handoffs, fewer silos)
* Onboarding gets easier (docs, runbooks, clear standards)

---

# Practical “Scrum Success Scorecard” (simple but powerful)

If you had to judge in 60 seconds:

1. **Sprint Goal hit-rate** (not story points)
2. **Cycle time trend** (down = good)
3. **Escaped defects / incidents trend** (down = good)
4. **WIP & blocker aging** (low/fast = good)
5. **Retro actions completed** (yes/no, consistent)

---
---

# How to gauge a Scrum team’s success (Senior Lead way)

## A) Value outcomes (most important)

**Ask:** Are we delivering meaningful outcomes, not just closing tickets?
**Measures:**

* Stakeholder satisfaction / adoption
* Feature usage metrics, conversion, revenue impact (if applicable)
* Reduction in incidents, support tickets, or operational load
* Clear Sprint Goals achieved consistently

---

## B) Predictability & reliability

**Ask:** Can we make commitments and meet them sustainably?
**Signals:**

* Sprint Goals met most sprints (even if scope changes)
* Stable throughput over time (not extreme spikes)
* Low “carry-over” / spillover stories
* Work gets to “Done” (including testing + release), not “almost done”

---

## C) Flow efficiency (how fast work moves)

**Ask:** Is work flowing smoothly from “ready” → “done”?
**Metrics that actually help:**

* **Cycle time** (start → done)
* **Lead time** (request → done)
* **WIP** (too high = congestion)
* Blocker age, time-to-unblock
* % work stuck in review/QA environments

> Healthy teams optimize **cycle time + WIP**, not velocity.

---

## D) Quality & engineering health

**Ask:** Are we shipping without breaking production?
**Signals:**

* Low escaped defects
* Incident rate trending down
* Good test signal (unit/integration/contract where relevant)
* Clean rollback / feature flags / safe releases
* Tech debt is managed (not ignored)

---

## E) Team maturity & continuous improvement

**Ask:** Are retros actually changing things?
**Signals:**

* Retro actions are implemented and reviewed
* Less repeated pain (fewer recurring blockers/incidents)
* Collaboration improves (less handoffs, fewer silos)
* Onboarding gets easier (docs, runbooks, clear standards)

---

# Practical “Scrum Success Scorecard” (simple but powerful)

If you had to judge in 60 seconds:

1. **Sprint Goal hit-rate** (not story points)
2. **Cycle time trend** (down = good)
3. **Escaped defects / incidents trend** (down = good)
4. **WIP & blocker aging** (low/fast = good)
5. **Retro actions completed** (yes/no, consistent)

---

## Interview-ready one-liner

> “A Scrum team is successful when it reliably delivers valuable increments every sprint with improving cycle time, stable predictability, and high quality—validated in reviews, sustained through healthy WIP and fast unblock, and improved via retros with measurable actions.”

