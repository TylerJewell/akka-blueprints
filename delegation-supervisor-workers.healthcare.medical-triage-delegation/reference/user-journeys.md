# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a question and watch parallel specialist synthesis

**Preconditions:** service running on `http://localhost:9155/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a medical question and Submit.
2. Observe the new case row via SSE.

**Expected:** the case progresses `TRIAGING → IN_REVIEW → RESPONDED` within ~90 s. The expanded row shows a `SymptomsAssessment` (urgency indicator + 2–5 identified symptoms), a `MedicationGuidance` (relevant OTC options), a `CareRecommendation` (recommended pathway + next steps), and a synthesised summary. The three specialist outputs arrive close together because the specialists ran in parallel. The `hotlPending` badge appears on the row after the response is recorded.

## J2 — Specialist timeout degrades the case

**Preconditions:** `SymptomsSpecialist` step timeout set to 1 s (test override).

**Steps:**
1. Submit a question.
2. Watch the case.

**Expected:** the `symptomsStep` times out, the workflow routes to `degradeStep`, and the TriageCoordinator synthesises from the MedicationGuidance and CareRecommendation outputs alone. The case enters `DEGRADED`; the summary's final sentence notes the missing symptoms assessment. No infinite retry. The `failureReason` field names the timed-out specialist.

## J3 — Safety guardrail blocks a diagnosis claim

**Preconditions:** TriageCoordinator returns a response with an explicit diagnosis claim (test fixture using a mock response with `"you have influenza"`).

**Steps:**
1. Submit the fixture question.
2. Watch the case.

**Expected:** `guardrailStep` flags the synthesised content; the workflow calls `block`; the case enters `BLOCKED` with a `failureReason` describing the rejected pattern. The case is never displayed as `RESPONDED` in the App UI; its row shows the BLOCKED pill with the failure reason in the expanded view.

## J4 — Human-on-the-loop flag appears and can be acknowledged

**Preconditions:** at least one `RESPONDED` case in the list.

**Steps:**
1. Observe the "Pending review" badge on a RESPONDED row.
2. Call `POST /api/triage/{id}/acknowledge-hotl` (or use the button in the expanded row).
3. Observe the badge.

**Expected:** before acknowledgement, `hotlPending` is `true` and the UI shows the badge. After acknowledgement, `HotlAcknowledged` is emitted, `hotlPending` flips to `false`, and the badge disappears from the row without a page refresh (SSE update).

## J5 — Compliance score appears beside a responded case

**Preconditions:** at least one `RESPONDED` case without a `complianceScore`.

**Steps:**
1. Wait for `ComplianceSampler` to run (every 5 minutes), or trigger it directly.
2. Observe the case row.

**Expected:** the case gains a `complianceScore` (1–5) and a `complianceRationale`; the App UI expanded row shows both. Response delivery was never blocked by the compliance sampling (non-blocking control).

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `CaseSimulator` drips a question from `medical-questions.jsonl` every 60 s; each becomes a case that flows through the full pipeline. The App UI is non-empty on first load and shows a variety of case statuses as the canned questions are processed.
