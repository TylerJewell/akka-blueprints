# User journeys — impact-study-runner

## J1 — Submit a request and reach the approval gate

**Preconditions:** Service running on declared port (`http://localhost:9684/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded request `REQ-2026-001 Lakeside Wind 150 MW` has a matching `src/main/resources/sample-data/load-flow/REQ-2026-001.json` file.

**Steps:**
1. Open `http://localhost:9684/` → App UI tab.
2. From the **Pick a seeded request** dropdown, pick `REQ-2026-001 Lakeside Wind 150 MW`.
3. Click **Run study**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `RUNNING_LOAD_FLOW` within 1 s more.
- Within ~20 s the card reaches `LOAD_FLOW_DONE`. The right pane shows the bus-voltage table with ≥ 4 rows and the line-flow table with ≥ 2 rows; each row has non-empty busId/lineId and numeric voltage/flow values.
- Within ~20 s more the card reaches `CONTINGENCY_DONE`. The right pane shows ≥ 1 violation with `isBinding: true` highlighted red and ≥ 4 voltage-check rows.
- Within ~20 s more the card reaches `AWAITING_APPROVAL`. The right pane shows the full `StudyReport` — executive summary, at least one section per binding violation — plus an eval score chip and the approve/reject controls.
- Total elapsed time to reach `AWAITING_APPROVAL`: ≤ 60 s.

## J2 — Engineer approves the study

**Preconditions:** A study from J1 is in `AWAITING_APPROVAL`.

**Steps:**
1. In the right pane, enter an optional approval note (e.g. `Binding violation acknowledged; mitigation plan attached.`).
2. Click **Approve**.

**Expected:**
- The card status transitions to `APPROVED` within 1 s.
- `GET /api/studies/{id}` returns `"status": "APPROVED"` and `"approvalNote"` populated with the entered text (or `null` if omitted).
- The approve/reject controls disappear from the right pane.
- The card border returns to the default (no amber highlight).

## J3 — Engineer rejects the study

**Preconditions:** A study from J1 is in `AWAITING_APPROVAL`.

**Steps:**
1. In the right pane, enter a rejection reason (e.g. `N-1 criterion not met — binding violation on LINE-202 requires mitigation before approval.`).
2. Click **Reject**.

**Expected:**
- The card status transitions to `REJECTED` within 1 s.
- `GET /api/studies/{id}` returns `"status": "REJECTED"` and `"approvalNote"` populated with the rejection reason.
- The card border highlights orange.
- The approve/reject controls disappear from the right pane.

## J4 — Low eval score flags the approval card

**Preconditions:** Mock LLM mode. The mock's `draft-report.json` includes one entry that sets `meetsN1Criterion: true` when the paired `ContingencyResult` contains a binding violation.

**Steps:**
1. Submit seeded requests until a study card's eval score chip shows ≤ 2 (the mock's `seedFor` selection cycles through the deliberately incorrect entry).
2. Observe the approval card for that study.

**Expected:**
- The study reaches `AWAITING_APPROVAL`.
- The eval score chip shows ≤ 2 and the rationale reads `"N-1 criterion parity failed: report.meetsN1Criterion is true but ContingencyResult contains binding violation CONT-LINE-xxx."`.
- The approval card has a red score border, signalling to the engineer that the automated check found a discrepancy.
- The approve/reject controls are still present — the eval score is informational, not a blocker.

## J5 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded request.
2. Wait for `AWAITING_APPROVAL`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the studyId.

**Expected:**
- The LOAD_FLOW task's log entries show only `computeBusVoltages` and `computePowerFlows` calls.
- The CONTINGENCY task's log entries show only `runN1Analysis` and `checkVoltageThresholds` calls.
- The DRAFT task's log entries show only `formatFinding` and `assembleExecutiveSummary` calls.
- No cross-phase calls appear.
- The order of tasks in the log is LOAD_FLOW → CONTINGENCY → DRAFT. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J6 — Request with no matching load-flow data

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/load-flow/<custom-slug>.json` exists for the user's request.

**Steps:**
1. In the App UI, type a custom request ID (e.g. `REQ-CUSTOM-999`) into the request-ID field.
2. Click **Run study**.

**Expected:**
- `LoadFlowTools.computeBusVoltages` returns an empty list.
- The agent's LOAD_FLOW task returns a `LoadFlowResult` with `busReadings = []` and `lineFlows = []`.
- The workflow advances to `contingencyStep`. The CONTINGENCY task returns a `ContingencyResult` with `violations = []`, `voltageChecks = []`, and `n1ContingenciesTested = 0`.
- The workflow advances to `draftStep`. The DRAFT task returns a `StudyReport` with `executiveSummary = "(no contingency data)"` and `sections = []`.
- The eval score chip shows 1 (no checks can pass on an empty result) with rationale `"no contingency data to evaluate"`.
- The workflow reaches `AWAITING_APPROVAL`; nothing crashes; the empty study is honestly empty.
- The engineer can reject it with reason `"no load-flow data available for this request ID"`.
