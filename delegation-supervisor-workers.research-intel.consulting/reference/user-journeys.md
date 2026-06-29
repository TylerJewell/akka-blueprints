# User journeys

Acceptance journeys. The blueprint generated correctly when each passes against the running service on `http://localhost:9895/`.

## J1 — Routine engagement is delegated and delivered

**Preconditions:** service running; a model provider configured (or mock).

**Steps:**
1. In the App UI tab, submit a routine brief, e.g. "Compile a competitor scan of EU industrial heat-pump vendors."
2. Watch the SSE list.

**Expected:**
- A new engagement appears in `RECEIVED`, then `ROUTED` with `route = DELEGATE` and a `complexityScore` below 0.5.
- It moves through `RESEARCHING` to `DELIVERED` with `assignedTo = junior` and a non-empty `deliverableContent` (a research brief).

## J2 — High-stakes engagement is handed off and delivered

**Preconditions:** service running.

**Steps:**
1. Submit a high-stakes brief, e.g. "Advise on the regulatory exposure of acquiring a competitor in a concentrated market."
2. Watch the SSE list.

**Expected:**
- The engagement routes with `route = HANDOFF` and `complexityScore` at or above 0.5.
- It moves through `CONSULTING` to `DELIVERED` with `assignedTo = senior` and a recommendation in `deliverableContent` that names risks and carries the advisory disclaimer.

## J3 — Compliance reviewer records a non-blocking review

**Preconditions:** a `DELIVERED` senior recommendation from J2.

**Steps:**
1. On that engagement's card, fill the compliance-review form (reviewer, verdict, notes).
2. Submit (`POST /api/engagements/{id}/compliance`).

**Expected:**
- The engagement moves to `COMPLIANCE_REVIEWED` (verdict PASS) or `FLAGGED` (verdict FLAG).
- `complianceReviewer`, `complianceVerdict`, `complianceNotes`, and `complianceReviewedAt` are populated. Delivery was not blocked waiting for the review.

## J4 — Every routing decision carries an eval score

**Preconditions:** at least one routed engagement.

**Steps:**
1. Inspect any engagement that has reached `ROUTED` or later.

**Expected:**
- `evalScore` is non-null on every routed engagement, recorded by `RoutingEvalConsumer` from the `EngagementRouted` event, and shown beside the routing rationale in the UI.

## J5 — Background load from the simulator

**Preconditions:** service running, no UI interaction.

**Steps:**
1. Leave the App UI tab open without submitting anything.

**Expected:**
- Every 30 seconds `RequestSimulator` drips a brief from `sample-events/engagements.jsonl`; new engagements appear and flow through routing and delivery on their own.

## J6 — Deliverable guardrail blocks weak output

**Preconditions:** service running.

**Steps:**
1. Submit a brief crafted to elicit a thin response (or rely on a mock entry tuned to fail the bar).

**Expected:**
- The before-agent-response guardrail rejects a deliverable that is too short, omits the disclaimer, or makes an unsupported guarantee; the workflow records the failed delivery rather than persisting unreviewed weak output.
