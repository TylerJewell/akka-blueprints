# User journeys — data-processing-planner

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on port 9394; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9394/`. App UI tab is visible.
2. In the Pipeline description field, type "Crawl the raw-events S3 prefix to discover schema, run an EMR step to de-duplicate records on event_id, and write the output to the processed/ prefix." In the Target dataset field, type "raw-events". Click Submit.
3. A new pipeline run card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to RUNNING via SSE.
- Within ~4 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A job ledger with a non-empty job plan (3–6 steps) and a `currentDispatch` field that is null at completion.
  - A run ledger with 3–6 entries spanning at least two engine types. Every entry has a non-empty `scrubbedOutput`.
  - A `PipelineOutput` with a 60–120 word summary, a non-empty `outputLocation`, a positive `rowsProcessed`, and a positive `totalCostUsd`.

## J2 — Cost guardrail blocks an over-budget dispatch

**Preconditions:** As J1.

**Steps:**
1. Submit the pipeline "Run a large-scale EMR cluster job to reprocess the entire data lake from 2020 onwards."

**Expected:**
- The PlannerAgent's first `DECIDE` proposes a `JobDispatch` with `estimatedCostUsd > 5.00` (e.g., a large EMR cluster step).
- The guardrail step rejects the dispatch; a `JobStepBlocked` entry appears on the run ledger with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker = "estimated cost 12.00 exceeds ceiling 5.00"`.
- The planner either revises to a smaller step within budget and the run completes, OR exhausts the replan budget and the run ends in `FAILED` with a clear `failureReason`. Either ending is acceptable; what matters is that the over-budget step never executes.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any pipeline that the simulator has not yet processed.
2. While the run status is RUNNING (within the first ~15 seconds), click **Cancel pipeline** in the operator pane and enter a reason such as "Investigating unexpected cost spike."
3. Observe the in-flight step completes.

**Expected:**
- The in-flight `RunEntry` is recorded normally (the workflow does not abort mid-step).
- The next loop iteration reads the halt flag, exits the loop, and emits `PipelineRunHaltedOperator`.
- Run status moves to `HALTED`. `haltReason` is populated with the operator's stated reason.
- The operator pane's `HALTED` pill and timestamp reflect the state in real time via the `control-update` SSE event.
- Other runs already enqueued do not start new dispatches until the operator clicks **Resume**.

## J4 — Secret sanitizer scrubs an exposed credential

**Preconditions:** As J1, with the fixture entry in `sample-data/job-fixtures.jsonl` whose output contains the literal substring `AKIAIOSFODNN7EXAMPLE` (the AWS documentation example access key). This fixture is exercised when the JobExecutorAgent processes a Glue crawler or EMR step that matches it.

**Steps:**
1. Submit "Find any stale AWS access credentials in the legacy pipeline job logs and summarise."

**Expected:**
- The PlannerAgent dispatches a job step that exercises the credential-containing fixture.
- The `JobExecutorAgent` returns output containing the literal `AKIAIOSFODNN7EXAMPLE`.
- The sanitize step replaces it with `[REDACTED:aws-access-key]` before the entry is recorded.
- The `RunEntry.scrubbedOutput` does NOT contain `AKIAIOSFODNN7EXAMPLE` anywhere.
- The PlannerAgent's next prompt (visible in subsequent `DECIDE` calls) sees only the redacted form.
- The final `PipelineOutput.summary` does not contain the literal key string.
- The UI's expanded run ledger renders the redacted span in italics with a tooltip showing `aws-access-key`.

## J5 — Stuck run auto-fails

**Preconditions:** As J1, with `application.conf` test override `stuck.threshold-minutes = 1`.

**Steps:**
1. Submit a pipeline and configure the planner (via prompt) to loop without progress (e.g., keep revising the plan without proposing any dispatch). Easier alternative: disable the LLM provider so calls time out.

**Expected:**
- After 1 minute of `RUNNING` without a completed step, `StuckPipelineMonitor` calls `PipelineEntity.timeoutFail`.
- The workflow's next `decideStep` reads `status = STUCK` and exits with `PipelineRunFailedTimeout`.
- Run status moves to `STUCK`. `failureReason` is `"stuck: no progress after 1m"`.
