# PROPOSAL-001: Non-Droppable Work via Structured Rejection + Arbiter-Owned State Machine

**Status:** Parked — Phase 3 (post-validation)
**Filed:** 2026-02-11
**Author:** Project lead (via GPT spec)
**When to revisit:** After P0 fixes ship, after 5 humans validate the base architecture, after framework-agnostic API is designed (ADR-011). This feature extends the Arbiter's role and should be built on a stable, proven foundation.

**Why not now:** This adds significant complexity to the Arbiter and bus protocol. Building it on top of unfixed P0 issues would compound the problems. The base state machine needs to work correctly before we add workflow states.

**Why it matters:** This is the bridge from "security governance" to "institutional cognition." It's what makes the 3-agent model feel like an organization, not a gate. It's also directly tied to the productivity framing (ADR-012) — handling pivot storms, ensuring no thread is lost.

**Dependencies:**
- P0 fixes (done ✅)
- Framework-agnostic bus protocol spec (ADR-011, not started)
- Stable Arbiter API
- Human validation that the base pattern has value

---

## Proposal: Non-Droppable Work via Structured Rejection + Arbiter-Owned State Machine

### Problem (observed in Nyx + our governance layer)

When a specialist (CISO, finance, logistics, etc.) rejects a request, the work can "fall on the floor" because the system treats rejection as terminal. In real orgs, rejection produces *routing*: ask for info, re-scope, reassign, defer, or escalate.

### Goal

Ensure **every request ends in an Arbiter-owned terminal state** (approved/executed/closed/deferred/waiting), and **no specialist can terminate work** without providing a structured outcome that enables next steps.

---

## Design Summary

### Key rule

**Only the Arbiter can set a task to a terminal state** (`CLOSED`, `DEFERRED`, `APPROVED`, `EXECUTED`). Specialists may only return one of a small set of **structured outcomes**.

### Specialist outcomes (finite)

Specialist response must set `outcome` ∈:

1. `NEEDS_INFO` – missing inputs; provide questions/required artifacts
2. `OUT_OF_SCOPE` – wrong domain; suggest specialist(s) to route to
3. `BLOCKED` – dependency required; specify dependency + owner
4. `TOO_COSTLY` – infeasible under constraints; propose cheaper alt / MVP
5. `POLICY_VIOLATION` – violates rule; cite rule + compliant alternatives
6. `LOW_CONFIDENCE` – uncertainty too high; specify what evidence would raise confidence
7. `APPROVE` – specialist approves (may include conditions)

> Note: "REJECT" is not allowed as a freeform terminal. All "no" must be one of the above.

---

## Task State Machine (Arbiter-owned)

### States

* `OPEN` – captured
* `ASSIGNED` – assigned to specialist
* `IN_REVIEW` – specialist working
* `REJECTED_WITH_REASON` – specialist returned structured non-approve
* `ESCALATED` – needs Arbiter decision / routing
* `REASSIGNED` – reassigned to another specialist
* `WAITING_ON_USER` – blocked on user info
* `BLOCKED` – blocked on internal dependency
* `DEFERRED` – parked intentionally with revisit trigger
* `APPROVED` – approved by Arbiter
* `EXECUTED` – executed (token consumed / command run)
* `CLOSED` – closed intentionally (won't be revisited)

### Invariant

Every task must always have:

* `owner` (who acts next)
* `state`
* `next_action` (what happens next)
* `unblock_condition` (what would move it forward)

---

## Routing Policy (Arbiter logic)

Given a `SpecialistResponse`:

* `NEEDS_INFO` → `WAITING_ON_USER`
  * Arbiter generates a short question list to user/operator.
  * When answers arrive, task returns to the same specialist.
* `OUT_OF_SCOPE` → `REASSIGNED`
  * Arbiter assigns to suggested specialist; if none exists, assigns to "generalist/operator".
* `BLOCKED` → `BLOCKED`
  * Arbiter creates dependency task(s) and assigns them.
  * Original task auto-resumes when dependencies complete.
* `TOO_COSTLY` → `ESCALATED`
  * Arbiter chooses: accept defer, approve MVP alternative, or request finance review.
* `POLICY_VIOLATION` → `ESCALATED`
  * Arbiter either closes as "not allowed" or routes to compliant alternatives / exception process.
* `LOW_CONFIDENCE` → `REASSIGNED` or `ESCALATED`
  * Route to research/testing specialist or request evidence.
* `APPROVE` → Arbiter may mark `APPROVED` and proceed to token issuance/execution pipeline.

---

## Message Schema (fits current Pydantic style)

### 1) SpecialistResponse (new or extend existing)

```json
{
  "task_id": "uuid",
  "specialist": "ciso|finance|logistics|integrations|operator",
  "outcome": "NEEDS_INFO|OUT_OF_SCOPE|BLOCKED|TOO_COSTLY|POLICY_VIOLATION|LOW_CONFIDENCE|APPROVE",
  "summary": "short natural language",
  "requests": ["question 1", "question 2"],
  "suggested_specialists": ["finance", "logistics"],
  "dependencies": [{"task": "do X", "owner": "operator"}],
  "alternatives": [{"option": "MVP approach", "delta": "..."}],
  "policy_refs": [{"id": "POL-001", "reason": "..." }],
  "confidence": 0.0,
  "evidence_needed": ["run test A", "collect data B"],
  "conditions": ["only if budget < $500", "only in sandbox"]
}
```

### 2) ArbiterDecision (new)

```json
{
  "task_id": "uuid",
  "decision": "REASSIGN|WAITING_ON_USER|CREATE_DEPENDENCY|DEFER|CLOSE|APPROVE",
  "assigned_to": "specialist-name",
  "created_tasks": ["uuid1","uuid2"],
  "note": "short rationale",
  "revisit_at": "timestamp|null"
}
```

### 3) Task (extend)

Add:
* `state`
* `owner`
* `unblock_condition`
* `history[]` (append-only list of SpecialistResponses + ArbiterDecisions)

---

## Repo-Specific Implementation Plan (minimal diff)

This is written for the packaged code path (`src/governance/*`).

### A) Models
* Add `SpecialistOutcome` enum + `SpecialistResponse` model (likely in `src/governance/models/decision.py` or a new `models/workflow.py`)
* Extend existing message schema validation in `src/governance/bus/*` to accept these new messages

### B) Orchestrator changes
In `src/governance/bus/orchestrator.py`:
* When a specialist outputs a non-approve response, write it as `SpecialistResponse` into a queue the Arbiter consumes
* Enforce: specialist outputs must validate against enum; reject/flag invalid "freeform reject"

### C) Arbiter routing engine
In `src/governance/arbiter/engine.py` (or a new `arbiter/router.py`):
* Implement `route_specialist_response(response) -> ArbiterDecision`
* Update task state, owner, and create dependency tasks as files/messages
* Guarantee that every response results in a task state transition and owner assignment

### D) Operator intake handling
In `src/governance/operator/interceptor.py`:
* When user provides answers to `NEEDS_INFO`, attach to task and requeue to the original specialist

### E) Audit
In `src/governance/audit/*`:
* Log each `SpecialistResponse` and `ArbiterDecision` as separate audit events

---

## Acceptance Criteria

### Scenario 1: NEEDS_INFO
* Create task → assign to CISO → CISO returns `NEEDS_INFO` with 3 questions
* ✅ Task enters `WAITING_ON_USER` with owner `operator/user`
* ✅ When answers supplied, task re-enters `ASSIGNED` to CISO

### Scenario 2: OUT_OF_SCOPE
* Specialist returns `OUT_OF_SCOPE` suggesting "finance"
* ✅ Arbiter reassigns to finance; task not closed

### Scenario 3: BLOCKED dependency
* Specialist returns `BLOCKED` with dependency "gather logs" owner "operator"
* ✅ Arbiter creates dependency task; parent task becomes `BLOCKED`
* ✅ When dependency closed, parent returns to `ASSIGNED` to original specialist

### Scenario 4: POLICY_VIOLATION
* Specialist returns `POLICY_VIOLATION` with policy id and alternatives
* ✅ Arbiter either closes with "not allowed" OR routes alternative to a specialist
* ✅ No silent drop

### Invariant check
* ✅ There is no path where a task is rejected and disappears without an ArbiterDecision.
