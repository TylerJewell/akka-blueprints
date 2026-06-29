# User journeys — mle-pipeline

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9762/`. App UI tab is visible.
2. In the form, enter dataset reference `credit-risk-schema.json` and objective `binary classification`. Click Submit.
3. A new run card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~5 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A pipeline ledger with a stages list of 4–6 steps and `activeDispatch` that is null at completion.
  - An evaluation ledger with 4–8 entries. At least one entry has `specialist = DATA_PROFILER`, at least one has `specialist = MODEL_EVALUATOR` with a non-null `MetricBundle` and `gateOutcome = PASSED`.
  - A `ModelReport` with a 60–120 word summary, a final `MetricBundle` with accuracy ≥ 0.80, and an ordered list of stage names.

## J2 — Eval gate blocks under-performing model

**Preconditions:** As J1. Configure `model-evaluator.json` mock to return a `MetricBundle` with accuracy = 0.73 on the first attempt (gate fails) and accuracy = 0.84 on the second attempt (gate passes).

**Steps:**
1. Submit a pipeline run with objective `binary classification`.

**Expected:**
- On the first `EVALUATE_MODEL` dispatch, the evaluator returns accuracy = 0.73 and `gateOutcome = FAILED`.
- The workflow's `gateStep` blocks the subsequent `PROMOTE` dispatch; a `StageGateFailed` entry appears on the evaluation ledger with `verdict = GATE_FAILED` and `gateOutcome = FAILED` and a `blocker` naming the failed threshold.
- The Planner replans (e.g., requests retraining with adjusted hyperparameters).
- On the second `EVALUATE_MODEL` dispatch, accuracy = 0.84 and `gateOutcome = PASSED`.
- The run proceeds to `PROMOTE` and ends in `COMPLETED`.
- The evaluation ledger contains two `MODEL_EVALUATOR` entries — the first with `GATE_FAILED`, the second with `PASSED`.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any run that the simulator has not yet enqueued.
2. While the run status is EXECUTING (within the first ~10 seconds), click **Halt new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight stage completes.

**Expected:**
- The in-flight `EvalEntry` is recorded normally (the workflow does not abort mid-stage).
- The next loop iteration reads the halt flag, exits the loop, and emits `RunHaltedOperator`.
- Run status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- Other runs already in the queue do not start their workflows until the operator clicks **Resume**.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.

## J4 — Drift watch raises an alert on a completed run

**Preconditions:** As J1. Baseline for `credit-risk-schema.json` in `metrics-baseline.json` has `auc = 0.91`. The mock evaluator returns `auc = 0.79` for the run in this journey (delta = 0.12, exceeds threshold of 0.10).

**Steps:**
1. Submit a run that produces a `ModelReport` with AUC = 0.79 (achieved via the mock evaluator fixture).
2. Wait for the run to complete (status = COMPLETED).
3. Wait up to 5 minutes for `DriftWatchMonitor` to fire (or advance the clock in the test config).

**Expected:**
- `DriftWatchMonitor` reads the completed run's MetricBundle, calls `DriftEvaluator.check`, and detects `|0.79 - 0.91| = 0.12 > 0.10`.
- `PipelineRunEntity.raiseDriftAlert` is called; a `DriftAlertRaised` event is emitted.
- `PipelineRun.driftAlerts` contains one entry with `metricName = "auc"`, `baseline = 0.91`, `observed = 0.79`, `delta = 0.12`.
- `GET /api/alerts` returns at least one alert matching the above.
- The run card in the UI shows a drift warning badge.
- The SSE channel emits a `drift-alert` event with the correct payload.
- The run status remains `COMPLETED` — the alert is informational and does not alter the run lifecycle.

## J5 — Stuck run auto-fails

**Preconditions:** As J1, with `application.conf` test override `stuck.threshold-minutes = 2`.

**Steps:**
1. Submit a run and configure the planner (via prompt or mock) to loop without dispatching any stage (e.g., emit `Replan` indefinitely). Easier alternative: disable the mock so every agent call times out.

**Expected:**
- After 2 minutes of `EXECUTING` without a recorded `EvalEntry`, `StaleRunMonitor` calls `PipelineRunEntity.timeoutFail`.
- The workflow's next `decideStep` reads `status = STUCK` and exits with `RunFailedTimeout`.
- Run status moves to `STUCK`. `failureReason` is `"stuck: no stage progress after 2m"`.
