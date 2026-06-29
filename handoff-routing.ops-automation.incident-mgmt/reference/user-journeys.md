# User journeys — incident-management

## J1 — Infrastructure incident: end-to-end handoff to InfraSpecialist

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `IncidentFeeder` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated report seeded as infrastructure-flavoured to drop.

**Expected:**
- The incident appears with status `RECEIVED` and within ~1 s transitions to `ENRICHED`. The enriched title is visible; the host group and service tier badges are populated.
- Within ~10 s the classification block shows `category = INFRASTRUCTURE`, `confidence = high`, and a one-sentence reason. The status pill changes to `CLASSIFIED` then `ROUTED_INFRA`.
- Within ~5 s of routing, the right column populates with the `InfraSpecialist` plan: a summary (two sentences max), a details block, and an action chip (typically `RESTART_SERVICE` or `SCALE_DEPLOYMENT`).
- The guardrail block shows a green "allowed" check.
- The status pill transitions to `RESOLVED`. `finishedAt` is set.
- Within ~10 s of the classification decision, the routing score chip shows a number 1–5 with a rationale.

## J2 — Application incident: end-to-end handoff to AppSpecialist

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the feeder to drop an application-flavoured report (the seeded JSONL covers the three categories in rotation).

**Expected:**
- Classification emits `category = APPLICATION`. Status `ROUTED_APP`.
- `AppSpecialist` returns a `RemediationPlan` with a concrete action (likely `ROLLBACK_DEPLOY` or `INFO_PROVIDED`).
- Guardrail allows. Status `RESOLVED`.
- The `InfraSpecialist` is never invoked for this incident — no infra plan appears in any surface.

## J3 — Ambiguous incident: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner alert.

**Steps:**
1. Wait for the ambiguous report to drop.

**Expected:**
- Classification emits `category = AMBIGUOUS` with `confidence = low`.
- The workflow terminates immediately with `IncidentEscalated`. Status `ESCALATED`.
- Neither specialist is invoked; the right column shows a muted "Escalated — no specialist invoked" block.
- `escalationReason` is populated with the classification reason.
- A `RoutingScored` event still fires; the score chip appears (typically high, since the classifier was right to refuse).

## J4 — Guardrail blocks an out-of-policy remediation plan

**Preconditions:** The seeded JSONL includes one report engineered to produce a peak-hours critical-host restart (and the mock-responses file includes a matching trip-the-guardrail entry).

**Steps:**
1. Wait for the trip report to drop and be classified as `INFRASTRUCTURE`.
2. Wait for `InfraSpecialist` to return a plan proposing a restart during peak hours on a critical-tier host.

**Expected:**
- Status reaches `PLAN_DRAFTED`.
- The guardrail block shows a red badge with the violations (`peak-hours-restart`, `critical-host-no-authority`).
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked plan (not executed) and an Unblock button.

## J5 — Operator unblocks a previously-blocked incident

**Preconditions:** An incident in `BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked incident.
2. Enter a note ("change window opened — CHG-5512 approved").
3. Confirm.

**Expected:**
- A `POST /api/incidents/{id}/unblock` is sent.
- The incident transitions to `RESOLVED`. The previously-blocked plan is now the published `RemediationPlan`.
- The guardrail block continues to show the original violation list (audit record preserved); a small note shows the operator's override and the change record reference.

## J6 — Routing score appears on every classified incident

**Preconditions:** Service running at least 60 s with the feeder on.

**Steps:**
1. Watch any new incident through the classification step.

**Expected:**
- Within ~10 s of `IncidentClassified`, the per-incident routing score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never changes the workflow's flow — a low score does not block the plan from being published.
