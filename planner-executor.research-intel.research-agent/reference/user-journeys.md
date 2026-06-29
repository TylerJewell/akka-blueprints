# User journeys — research-agent

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: query completes with a cited report

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9599/`. App UI tab is visible.
2. In the Research query field, type "Summarise the current regulatory landscape for AI transparency in the EU." Click Submit.
3. A new job card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to SEARCHING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A search ledger with a non-empty plan (3–6 steps) and a `currentDispatch` field that is null at completion.
  - A findings ledger with 3–6 entries. At least one entry has `searcher = WEB` and at least one has `searcher = DOCUMENT`. Every `CITED` entry shows a confidence value ≥ 0.25.
  - A `ResearchReport` with a 60–120 word summary and at least 2 citations. Every citation in the report traces back to a finding entry with `citationVerdict = CITED`.

## J2 — Citation eval forces a replan on an uncited finding

**Preconditions:** As J1. The document fixture `sample-data/docs/unsourced-overview.md` contains no author, URL, or ISBN (this fixture is included to exercise the UNCITED path).

**Steps:**
1. Submit "Explain the history of explainability in machine learning."

**Expected:**
- The planner dispatches a DOCUMENT search step that returns content from `unsourced-overview.md`.
- The citation evaluator marks the finding `UNCITED` with `confidence = 0.0` and a `noSourceReason`.
- The `FindingEntry` in the expanded UI shows the yellow `UNCITED` chip.
- The planner's next DECIDE_NEXT emits `Replan`, revising the strategy to attempt a WEB search for a sourced version of the same fact.
- The final report does not cite the uncited finding.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any query the simulator has not yet sent.
2. While the job status is SEARCHING (within the first ~10 seconds), click **Halt new dispatches** in the operator pane and enter a reason.
3. Observe the in-flight search step completes.

**Expected:**
- The in-flight `FindingEntry` is recorded normally — the workflow does not abort mid-step.
- The next loop iteration reads the halt flag and exits, emitting `JobHaltedOperator`.
- Job status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.
- Other jobs queued after the halt do not start until the operator clicks **Resume**.

## J4 — CITED finding with high confidence appears in the final report

**Preconditions:** As J1. The document fixture `sample-data/docs/eu-ai-act-transparency.md` carries an ISBN in its header and a source URL in its body.

**Steps:**
1. Submit "What does Article 13 of the EU AI Act require of high-risk AI system providers?"

**Expected:**
- The planner dispatches a DOCUMENT search step.
- The `DocumentSearchAgent` returns content including the document path `sample-data/docs/eu-ai-act-transparency.md`, the ISBN, and the source URL.
- The citation evaluator matches both the document-path signal and the URL signal; confidence is 0.75 or higher; `citationVerdict = CITED`.
- The `FindingEntry` in the expanded UI shows the green `CITED` chip with confidence displayed as a percentage.
- The final `ResearchReport.citations` list includes an entry whose `source` field references the document path and whose `label` names Article 13.
