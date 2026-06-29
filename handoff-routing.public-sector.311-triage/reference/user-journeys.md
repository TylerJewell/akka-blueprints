# User journeys — 311-triage

## J1 — Public-works request: end-to-end handoff to PublicWorksSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `RequestSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated request seeded as public-works-flavoured to drop (e.g. a pothole report).

**Expected:**
- The request appears with status `RECEIVED` and within ~1 s transitions to `SANITIZED`. The redacted subject is visible; any raw PII is replaced with category tokens.
- Within ~10 s the triage block shows `category = PUBLIC_WORKS`, `confidence = high`, and a one-sentence reason. The status pill changes to `TRIAGED`.
- The route verdict block shows a green "approved" check. Status transitions to `ROUTE_APPROVED` then `ROUTED_PUBLIC_WORKS`.
- Within ~5 s of routing, the right column populates with the `PublicWorksSpecialist` response: a `Re: …` subject, a 2–4 paragraph body, an action chip (typically `WORK_ORDER_CREATED` or `INFO_PROVIDED`), and a `public-works` department tag.
- The status pill transitions to `RESOLVED`. `finishedAt` is set.
- Within ~10 s of the triage decision, the triage score chip shows a number 1–5 with a rationale.

## J2 — Permits request: end-to-end handoff to PermitsSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a permits-flavoured request (the seeded JSONL covers the categories in rotation).

**Expected:**
- Triage emits `category = PERMITS_ZONING`. Status `ROUTE_APPROVED` → `ROUTED_PERMITS_ZONING`.
- Route guardrail approves.
- `PermitsSpecialist` returns a `DepartmentResponse` with a concrete answer (typically `PERMIT_INFO_PROVIDED` or `INSPECTION_SCHEDULED`).
- Status `RESOLVED`.
- `PublicWorksSpecialist` is never invoked for this request — no response from it appears anywhere.

## J3 — Ambiguous request: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous request to drop.

**Expected:**
- Triage emits `category = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `RequestEscalated`. Status `ESCALATED`.
- Neither specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block.
- `escalationReason` is populated with the triage reason.
- A `TriageScored` event still fires; the score chip appears (typically high, since the classifier was right to refuse).

## J4 — Route guardrail flags a low-confidence route

**Preconditions:** The seeded JSONL includes a borderline request that the triage agent classifies with low confidence.

**Steps:**
1. Wait for the borderline request to drop.
2. Observe the route guardrail step fire after triage.

**Expected:**
- Triage emits a decision with `confidence = low`.
- The route guardrail returns `RouteVerdict { approved: false, flags: ["low-confidence-route"] }`.
- Status transitions to `FLAGGED_FOR_REVIEW`. No specialist is invoked.
- The right column shows the flag detail block with the `low-confidence-route` flag and a Review button.
- `finishedAt` remains null.

## J5 — Triage reviewer resolves a flagged request

**Preconditions:** A request in `FLAGGED_FOR_REVIEW`.

**Steps:**
1. Click the Review button on the flagged request.
2. Enter a note ("Reviewed; route to public works confirmed by supervisor").
3. Click "Approve route".

**Expected:**
- A `POST /api/requests/{id}/review` is sent with `decision="approve"`.
- The workflow resumes at the route step. The request transitions `ROUTE_APPROVED` → `ROUTED_*` → `RESPONSE_DRAFTED` → `RESOLVED`.
- The route verdict block retains the original flags (audit record preserved); a note shows the reviewer's override.
- The specialist is invoked and the response is published.

## J6 — Triage score appears on every triaged request

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new request through the triage step.

**Expected:**
- Within ~10 s of `TriageDecided`, the per-request triage score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never changes the workflow's flow — a low score does not block the response from being published.
