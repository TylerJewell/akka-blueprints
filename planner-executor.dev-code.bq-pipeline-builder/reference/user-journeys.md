# User journeys — bq-pipeline-builder

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:<port>/`. App UI tab is visible.
2. In the Pipeline description field, type "Build a pipeline that loads raw GA4 events into a clean sessions table with session-level aggregates." Click Submit.
3. A new pipeline card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to BUILDING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A pipeline ledger with a non-empty build plan (3–8 steps) and a `currentDispatch` field that is null at completion.
  - A step ledger with 3–8 entries. At least one entry has `executor = SCHEMA_ANALYST`, at least one has `executor = SQL_COMPOSER`. Every entry has a non-empty `output`.
  - A `BuildManifest` with a 60–120 word summary and 3–5 artifact bullets, each tagged with the executor kind.

## J2 — DDL guardrail blocks a destructive dispatch

**Preconditions:** As J1.

**Steps:**
1. Submit the prompt "Clean up the sessions table by dropping and recreating it from scratch."

**Expected:**
- The planner's first `DECIDE_STEP` proposes a `SQL_COMPOSER` step whose text contains `DROP TABLE`.
- The guardrail step rejects the decision; a `StepBlocked` entry appears on the step ledger with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker` describing the rejected DDL pattern.
- The planner either replans to a non-destructive `TRUNCATE … INSERT` equivalent and completes, OR exhausts the replan budget and the pipeline ends in `FAILED` with a clear `failureReason`. Either outcome is acceptable; what matters is that the `DROP TABLE` statement never reaches `SqlComposerAgent`.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any pipeline description that the simulator has not yet seen.
2. While the pipeline status is BUILDING (within the first ~10 seconds), click **Halt new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight build step completes.

**Expected:**
- The in-flight `StepEntry` is recorded normally (the workflow does not abort mid-step).
- The next loop iteration reads the halt flag, exits the loop, and emits `PipelineHaltedOperator`.
- Pipeline status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- Pipelines already queued but not yet started do not dispatch new steps until the operator clicks **Resume**.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.

## J4 — CI gate blocks a malformed Dataform definition

**Preconditions:** As J1, with the mock fixture `dataform-modeler.json` containing an entry whose content is a SQLX definition with a missing `config` block.

**Steps:**
1. Submit "Model the sessions table as a Dataform SQLX definition and validate it."

**Expected:**
- The planner dispatches a `DATAFORM_MODELER` step.
- `DataformModelerAgent` returns a SQLX definition that omits the `config` block.
- `ValidationSuite.run(stepOutput)` detects the missing `config` block and returns `ValidationReport { passed: false, failures: ["SQLX definition missing config block in sessions.sqlx"] }`.
- The workflow records a `StepEntry` with verdict `CI_GATE_FAILED` and loops back to `proposeStep`.
- The planner sees `CI_GATE_FAILED` on its next `DECIDE_STEP` call and either proposes a corrected `DATAFORM_MODELER` step or replans.
- After three consecutive `CI_GATE_FAILED` verdicts on the same step, the failure budget is exhausted and the pipeline ends in `FAILED` with `failureReason` naming the step and the gate failure.

## J5 — Stuck pipeline auto-fails

**Preconditions:** As J1, with `application.conf` test override `stuck.threshold-minutes = 1`.

**Steps:**
1. Submit a pipeline and configure the planner (via prompt) to loop without progress (e.g., keep issuing the same failing `DATAFORM_MODELER` step in a way that the CI gate always fails and the replan budget is exhausted silently). Easier alternative: cut the LLM provider mid-pipeline so calls time out.

**Expected:**
- After 1 minute of `BUILDING` without progress, `StalePipelineMonitor` calls `PipelineEntity.timeoutFail`.
- The workflow's next `decideStep` reads `status = STUCK` and exits with `PipelineFailedTimeout`.
- Pipeline status moves to `STUCK`. `failureReason` is `"stuck: no progress after 1m"`.
