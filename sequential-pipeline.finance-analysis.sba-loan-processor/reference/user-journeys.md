# User journeys — sba-loan-processor

## J1 — Submit an application and complete the full officer-approval cycle

**Preconditions:** Service running on `http://localhost:9999/`; a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded application `app-001` (Sunrise Bakery LLC, $250,000 equipment loan) has a matching `src/main/resources/sample-data/applications/app-001.json`.

**Steps:**
1. Open `http://localhost:9999/` → App UI tab.
2. From the **Pick a seeded application** dropdown, pick `Sunrise Bakery — $250,000`.
3. Click **Process application**.
4. Watch the card progress in the live list.
5. When the card reaches `PENDING_REVIEW`, expand the right pane and click **Approve** with a note.

**Expected:**
- The new card appears with status `SUBMITTED` within 1 s, then `INTAKE_IN_PROGRESS` within 1 s more.
- Within ~25 s the card reaches `INTAKE_COMPLETE`. The right pane shows the credit profile panel with a credit score tier badge and business profile.
- Within ~25 s more the card reaches `UNDERWRITING_COMPLETE`. The right pane shows the underwriting analysis panel with NOI, LTV ratio, key risk factors, and mitigants.
- Within ~25 s more the card reaches `DECISION_MADE`, then `PENDING_REVIEW` within 2 s. The right pane shows the loan decision panel and the fairness eval score chip (≥ 3). The officer review panel (Approve / Deny buttons) is now active.
- The officer clicks **Approve**. Within 2 s the card transitions to `OFFICER_APPROVED`. The memo panel appears. Total elapsed time from submit to `PENDING_REVIEW`: ≤ 90 s.

## J2 — Phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `make-decision.json` includes one entry whose `tool_calls` array starts with a REPORT-phase tool (`formatDecisionRationale`) — the deliberately phase-violating entry, selected on the first iteration of every 3rd application.

**Steps:**
1. Submit any seeded application three times in a row (J1 steps × 3, stopping before officer review).
2. Watch the third submission's lifecycle in the network panel (`/api/loans/sse`).

**Expected:**
- On the third submission's `decisionStep`, the agent's first iteration calls `formatDecisionRationale`. `PhaseGuardrail` rejects it; a `GuardrailRejected{phase: "DECISION", tool: "formatDecisionRationale", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `ReportTools.formatDecisionRationale` — there is no log line from the tool body.
- The agent's second iteration calls `computeDscr` (correct phase). The application eventually reaches `PENDING_REVIEW` as in J1.
- The card in the App UI shows the small red dot. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Protected-attribute sanitizer prevents attribute leakage

**Preconditions:** Service running. The seeded application `app-003` includes `race: "Hispanic"` and `maritalStatus: "Married"` in its raw JSON.

**Steps:**
1. Submit `app-003` and wait for the application to reach `PENDING_REVIEW`.
2. Read the `LoanDecision.rationale` and `UnderwritingAnalysis.keyRiskFactors` from the right pane.
3. Inspect the service log filtered to the `underwriteStep` and `decisionStep` entries for `app-003`.

**Expected:**
- The decision rationale contains no reference to `Hispanic`, `Married`, `race`, `maritalStatus`, or any value from the `PROXY_TERMS` list.
- The log lines for the UNDERWRITE and DECISION tasks show that the instruction context passed to the model contained `"[REDACTED]"` for all protected-attribute fields.
- The fairness eval score chip shows ≥ 4 (proxy-language check passes; no proxy terms in output).

## J4 — Proxy-language in decision rationale flags eval score ≤ 2

**Preconditions:** Mock LLM mode. The mock's `make-decision.json` includes one entry whose `rationale` field contains the proxy term `"neighborhood stability"`. This entry is selected once in every five runs by the mock's `seedFor(applicationId)` modulo.

**Steps:**
1. Submit any seeded application five times. Stop before officer review on each.
2. Watch the live list until a card's fairness eval score chip shows ≤ 2.

**Expected:**
- The flagged application's right pane shows the fairness eval section with a score ≤ 2.
- The `flags` list contains `"proxy-language: 'neighborhood stability' detected in rationale"`.
- The rationale in the fairness eval section names the exact offending term.
- The other four applications in the run scored ≥ 4.
- The flagged application remains in `PENDING_REVIEW` — the fairness score is non-blocking; the officer still approves or denies.

## J5 — Officer denial produces no memo and closes the application

**Preconditions:** Service running. Any model provider. An application in `PENDING_REVIEW` state.

**Steps:**
1. Submit `app-002` and wait for `PENDING_REVIEW`.
2. In the officer review panel, click **Deny** with the note "Insufficient DSCR for requested amount."
3. Observe the card and the entity state.

**Expected:**
- Within 2 s the card transitions to `OFFICER_DENIED`. The `officerReview` field on the right pane shows the officer ID, outcome, and notes.
- The `memo` field on the row record is `null` — no `GENERATE_REPORT` task was called. The right pane does not show a memo panel.
- The SSE stream emits one `loan-update` event with `status: "OFFICER_DENIED"` and all other listening clients update the card simultaneously.
- The workflow is terminated. No further state transitions occur on this application.

## J6 — DSCR calculation correctness verified by fairness scorer

**Preconditions:** Service running. The seeded application `app-005` has `netOperatingIncome = 90000` and requires `annualDebtService = 100000` (DSCR < 1.25, so `DENY` is expected).

**Steps:**
1. Submit `app-005` and wait for `PENDING_REVIEW`.
2. Inspect the `LoanDecision.dscrResult` on the right pane.
3. Inspect the `fairnessEval` section.

**Expected:**
- `dscrResult.dscr` equals `0.90` (= 90000 / 100000).
- `dscrResult.meetsThreshold` is `false`.
- `LoanDecision.recommendation` is `DENY`.
- `LoanDecision.rationale` contains a phrase referencing the DSCR figure and the 1.25 threshold — the rationale completeness check passes.
- The fairness eval score chip shows ≥ 4 (all four checks pass). The DSCR correctness check confirms that `computeDscr` was called with the correct inputs from the UNDERWRITE phase.
