# User journeys — tau2-benchmark-agent

## J1 — Submit a web-navigation task and get a scored result

**Preconditions:** Service running on `http://localhost:9330/`; a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9330/` → App UI tab.
2. From the **Task category** dropdown, pick `web-navigation`.
3. Click **Load seeded example** to fill all task fields from the seeded corpus.
4. Click **Run task**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `EXECUTING` within 1 s. The right-pane detail shows the task description and step list.
- Within 60 s the card reaches `RESULT_RECORDED`. The right pane shows: an outcome badge (PASS / PARTIAL / FAIL), one step-output block per task step (all non-empty), a final answer, and a latency measurement.
- Within 1 s of `RESULT_RECORDED`, the card reaches `SCORED` and shows a score chip (1–5) plus a one-line rationale.
- The aggregate panel updates its pass-rate gauge and mean-score display.

## J2 — Evidence-thin result flags score 1

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `execute-benchmark-task.json` includes an evidence-thin entry with all empty `stepOutputs` and an empty `finalAnswer`. The mock selects this entry every 4th run (by seed).

**Steps:**
1. Submit the seeded web-navigation task four times in a row.
2. Watch the fourth submission's lifecycle in the run list.

**Expected:**
- The fourth run's `TaskResult` has all `StepOutput.output` strings empty and an empty `finalAnswer`.
- The `TaskScorer` scores this result 1 (zero criteria met).
- The rationale reads "Step outputs are empty; final answer is empty; result is not anchored in task execution."
- The card's border highlights red. The reviewer knows to inspect this run before drawing capability conclusions.
- The aggregate panel reflects the failed score in the mean-score calculation.

## J3 — Perfect result scores 5 and increments pass rate

**Preconditions:** Service running with the mock LLM selected. A mock entry exists with all steps non-empty, `finalAnswer` matching `referenceAnswer` (case-insensitive, trimmed), and `latencyMs` in (0, 120000].

**Steps:**
1. Submit the seeded tool-use task.
2. Wait for `SCORED`.
3. Inspect the aggregate panel.

**Expected:**
- The `TaskScorer` awards score 5.
- The rationale reads "All steps attempted with non-empty outputs; final answer matches reference answer; latency within bounds."
- The aggregate panel's pass-rate gauge increments and mean score reflects the new 5.
- The run card's border is not highlighted (score > 2).

## J4 — Agent timeout transitions run to FAILED

**Preconditions:** Service running. The `executeStep` timeout is 60 s.

**Steps:**
1. In a test environment, configure the mock LLM to delay its response beyond 60 s (or submit a real task with an API key and intentionally disconnect the model provider mid-call).
2. Submit any seeded task.
3. Wait for the run card to reach a terminal state.

**Expected:**
- After 60 s, `BenchmarkRunWorkflow.executeStep` times out. The workflow transitions the entity to `FAILED` via the error step.
- The run card shows status `FAILED` with a red pill.
- The right pane shows the task definition (from `request`) even though no `result` or `score` landed — the entity preserved the partial state.
- No `ResultRecorded` or `RunScored` event appears in the entity log.

## J5 — Aggregate breakdowns by category

**Preconditions:** Service running. At least two runs per category have reached `SCORED`.

**Steps:**
1. Submit two web-navigation, two tool-use, and two multi-step-reasoning tasks (using seeded examples or custom definitions).
2. Wait for all six to reach `SCORED`.
3. Inspect the aggregate panel.

**Expected:**
- The `GET /api/runs/aggregate` response lists totals and pass/partial/fail counts for each of the three categories.
- The per-category bar in the UI reflects the counts correctly.
- `passRate` in the aggregate equals the count of PASS outcomes divided by total SCORED runs (excluding FAILED runs).
- `meanScore` equals the arithmetic mean of all non-null `score.score` values across SCORED runs.
