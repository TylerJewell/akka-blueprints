# User journeys — cross-agency-case-handoff-mesh

## J1 — Intake-eligible case: end-to-end handoff to BenefitsReviewer

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `CaseSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated case seeded as intake-eligible to drop.

**Expected:**
- The case appears with status `RECEIVED` and transitions to `SCOPED` within ~1 s. The redacted summary is visible; the raw `fullPayload` is not.
- Within ~10 s the routing block shows `segment = INTAKE`, `confidence = high`, and a one-sentence rationale. Status transitions to `ROUTED` then `JURISDICTION_CHECK`.
- The jurisdiction verdict block shows a green check (`allowed=true`) and the `INTAKE-01` agency code. Status transitions to `IN_SEGMENT`.
- Within ~5 s of `IN_SEGMENT`, `IntakeAssessor` returns a `SegmentOutcome` with determination `REFERRED_TO_BENEFITS` and a 2–4 sentence summary note. Status transitions to `SEGMENT_COMPLETE` then `HANDOFF_PENDING`.
- The right column shows the Approve / Reject buttons. The status pill pulses amber.
- The operator clicks Approve, enters a note, and confirms. Status transitions to `HANDOFF_EMITTED`.
- The workflow immediately starts the benefits segment: the case re-enters `ROUTED`, `JURISDICTION_CHECK`, `IN_SEGMENT`, `SEGMENT_COMPLETE`, then `HANDOFF_PENDING` again.
- The operator approves the second handoff. Status transitions to `CLOSED` with `closedAt` set.

## J2 — Direct benefits renewal: single-segment close

**Preconditions:** Same as J1. The seeded JSONL includes a housing-aid renewal.

**Steps:**
1. Wait for the renewal case to drop.

**Expected:**
- Routing emits `segment = BENEFITS`. Status `ROUTED`.
- Jurisdiction guardrail clears. Status `IN_SEGMENT`.
- `BenefitsReviewer` returns `SegmentOutcome { determination: AWARD_ISSUED }`.
- Status reaches `HANDOFF_PENDING`. The right column shows Approve / Reject.
- Operator approves. Because this is the final segment, status transitions directly to `CLOSED`.
- The `IntakeAssessor` is never invoked for this case.

## J3 — Out-of-jurisdiction case: blocked before agent invocation

**Preconditions:** The seeded JSONL includes a case with a geographic region outside the agency mesh.

**Steps:**
1. Wait for the out-of-jurisdiction case to drop.

**Expected:**
- Routing emits `segment = INTAKE` or `segment = BENEFITS` with medium or high confidence.
- The jurisdiction guardrail returns `allowed=false` with violation `geographic-region-out-of-scope`.
- Status transitions immediately to `JURISDICTION_BLOCKED`. The centre column shows the jurisdiction verdict with the red violation badge.
- No agency agent (`IntakeAssessor` or `BenefitsReviewer`) is ever invoked. The right column shows a "Jurisdiction Blocked" block.

## J4 — Case-owner rejects the handoff

**Preconditions:** A case has reached `HANDOFF_PENDING` after a successful intake assessment.

**Steps:**
1. Observe a case in `HANDOFF_PENDING` (amber-pulsing status pill).
2. Click the Reject button.
3. Enter a reason ("Intake determination incomplete — missing income documentation").
4. Confirm.

**Expected:**
- `POST /api/cases/{id}/reject-handoff` is sent.
- Status transitions to `HANDOFF_REJECTED`. The right column shows a red "Transfer Rejected" block with the rejection reason and the case-officer's identifier.
- The `BenefitsReviewer` is never invoked. The intake outcome remains visible in the right column for audit.

## J5 — Unroutable case: short-circuits without invoking any agent

**Preconditions:** The seeded JSONL includes a case with an unrecognised program type.

**Steps:**
1. Wait for the unroutable case to drop.

**Expected:**
- Routing emits `segment = UNROUTABLE` with `confidence = low`.
- Status transitions to `UNROUTABLE`. The right column shows a muted "Unroutable — no agency segment" block with the routing rationale.
- Neither `IntakeAssessor` nor `BenefitsReviewer` is invoked. No HITL approval panel appears.
