# User journeys — warehouse-optimizer

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the attempt ceiling

**Preconditions:** Service running on port 9771; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9771/`. App UI tab is visible.
2. In the SQL Statement field, enter `SELECT * FROM orders WHERE customer_id = 42`. Set Objective to `reduce full-table scan on orders`. Click Submit.
3. A new request card appears with status `PROPOSING`.

**Expected:**
- Within 1 s, the first attempt's proposal appears.
- The guardrail verdict pill on attempt 1 reads `OK` (non-DDL proposal).
- Within 60 s of submission, either:
  - Attempt 1's evaluation is `APPROVE` and the request transitions to `APPROVED`, OR
  - Attempt 1's evaluation is `REVISE` with 1–3 bullets and the request transitions back to `PROPOSING`, with attempt 2 appearing shortly after; this continues until either an `APPROVE` or the retry ceiling is reached.
- On `APPROVED`, the terminal block shows the approved SQL and "best of N attempts."
- The expanded view shows every attempt's proposal, guardrail verdict, evaluator verdict, score, and notes.

## J2 — Halt at attempt ceiling

**Preconditions:** As J1, plus an override that forces the Evaluator to always return `REVISE` (test mode — submit the literal objective `"test-force-reject"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit the SQL `SELECT * FROM events` with objective `"test-force-reject"` and the default settings.

**Expected:**
- Request progresses `PROPOSING` → `EVALUATING` → `PROPOSING` → `EVALUATING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the request transitions to `REJECTED_FINAL` (not stuck in `EVALUATING`).
- The terminal block shows the highest-scoring proposal's SQL as the "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its proposal, guardrail OK verdict, REVISE evaluation, and score.
- `GET /api/requests/{id}` returns the full `OptimizationRequest` with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — DDL guardrail and DBA approval gate

**Preconditions:** As J1.

**Steps:**
1. Submit the SQL `ALTER TABLE orders ADD COLUMN loyalty_tier VARCHAR(20)` with objective `"add loyalty tier column"`.

**Expected:**
- Attempt 1's proposal contains the DDL statement. The guardrail records `verdict.passed = false`, `reasonCode = "DDL_DETECTED"`, with detail naming the detected keyword.
- The Evaluator is NOT called for attempt 1. The request transitions to `AWAITING_DBA` (amber border).
- An inline DBA decision form appears in the expanded view for attempt 1.
- Submit an Approve decision with a note. The request resumes: the workflow calls the Evaluator on the same proposal.
- The Evaluator's verdict and score appear. If `APPROVE`, the request transitions to `APPROVED`; if `REVISE`, the loop continues with attempt 2 (which will again pass through the guardrail check).
- The expanded view shows attempt 1 with the `DDL_DETECTED` amber pill, the recorded DBA decision, and the evaluator's verdict.

**Alternate path — DBA rejects:**
- Submit a Reject decision instead of Approve.
- The request transitions immediately to `REJECTED_FINAL` with `rejectionReason` reflecting the DBA's decision note.
- The Evaluator is never called.

## J4 — Eval-event timeline

**Preconditions:** At least one request has completed (any terminal state).

**Steps:**
1. Click the request card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per evaluated attempt, with `verdict`, `score`, and `ddlDetected` populated.
- The terminal transition (RequestApproved or RequestRejectedFinal) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/requests/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
