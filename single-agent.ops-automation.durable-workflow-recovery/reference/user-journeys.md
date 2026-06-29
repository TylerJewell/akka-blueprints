# User journeys — durable-workflow-recovery

## J1 — Register an ETL pipeline and receive a RESUME decision

**Preconditions:** Service running on declared port (`http://localhost:9719/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9719/` → App UI tab.
2. From the **Workflow type** dropdown, pick `ETL pipeline`.
3. Click **Load seeded example** to fill the workflow ID and owner team.
4. Click **Register execution**.

**Expected:**
- The new card appears in the live list with status `REGISTERED` within 1 s.
- The card transitions to `RUNNING` within 1 s as seeded checkpoint events replay. The checkpoint timeline fills in node by node (extract → transform → ...).
- After the configured stall timeout elapses with no further checkpoint, the card transitions to `STALLED`. The snapshot summary shows `stalledFor` duration.
- Within 30 s the card reaches `DECISION_RECORDED`. The right pane shows: a verdict badge (`RESUME`), the rationale paragraph, and a checkpoint status row for each of the 8 expected checkpoints. Every row has a non-empty `checkpointId`, `phase`, `outcome`, and `recommendedAction`.
- Within 5 s of `DECISION_RECORDED`, the card reaches `HEALTH_SCORED` and shows a health score chip (1–5) plus a one-line diagnosis.

## J2 — Payment-batch execution exhausts retries and receives ABORT

**Preconditions:** Service running with the mock LLM selected. The mock's `analyze-execution.json` includes an ABORT entry for the payment-batch workflow type.

**Steps:**
1. From the **Workflow type** dropdown, pick `Payment batch`.
2. Click **Load seeded example** (all 5 checkpoints in `FAILED` state in the seed).
3. Click **Register execution**.
4. Watch the card lifecycle in the browser's network panel via `/api/executions/sse`.

**Expected:**
- The card reaches `STALLED` after checkpoint replay completes.
- The agent returns a `ABORT` verdict because `failedCount / totalExpected > 50%`.
- The entity transitions to `ABORTED`. No `ResumeCommand` is dispatched — confirmed by the absence of any `RUNNING` transition after `ANALYZING`.
- The UI card's border turns red. The verdict badge shows `ABORT`. The checkpoint status table shows all checkpoints with `FAILED` outcome and a `recommendedAction` of "Mark as abandoned — execution will not restart."

## J3 — Latency-degraded execution receives health score 1

**Preconditions:** Mock LLM mode. The data-migration seed history has checkpoint `elapsedMs` values 4× the configured baseline for that workflow type.

**Steps:**
1. From the **Workflow type** dropdown, pick `Data migration`.
2. Click **Load seeded example** (includes the high-latency checkpoint history).
3. Click **Register execution**.
4. Wait for `HEALTH_SCORED`.

**Expected:**
- The decision lands with a `RESUME` verdict (only one checkpoint is PENDING; state is recoverable).
- The health score chip shows **1** and the diagnosis reads "Median checkpoint latency is 4× above the ETL baseline; execution is at risk of repeated stalls."
- The card border highlights red. The operator knows to investigate the upstream data source before resuming.

## J4 — Mid-checkpoint crash recovers from last COMPLETED checkpoint

**Preconditions:** Service running. Any model provider. ETL-pipeline seed used.

**Steps:**
1. Register the ETL-pipeline seed (J1 steps).
2. Wait for `DECISION_RECORDED`.
3. Inspect the checkpoint status table in the right pane.

**Expected:**
- The `cp-load-003` checkpoint (the one that was PENDING at crash time) has `outcome = PENDING` in the decision's `checkpointStatuses`, not `FAILED`.
- The `recommendedAction` for `cp-load-003` is "Retry from this checkpoint — prior state is durable."
- The preceding checkpoints (`cp-extract-001`, `cp-transform-002`) have `outcome = COMPLETED` and `recommendedAction = "No action — checkpoint is durable."`.
- The service log shows a `ResumeCommand` dispatched with `fromCheckpointId = cp-transform-002` (the last COMPLETED checkpoint before the pending one).

## J5 — ResponseValidator rejects malformed mock response

**Preconditions:** Mock LLM mode. The mock selects a malformed entry on the first iteration of every 3rd execution (a `checkpointStatus.checkpointId` that does not exist in the snapshot).

**Steps:**
1. Register three ETL-pipeline executions in sequence (J1 steps × 3).
2. Watch the third execution's lifecycle in the browser's network panel.

**Expected:**
- The third execution's `analyzeStep` receives a malformed decision from the mock agent.
- `ResponseValidator.validate(decision, snapshot)` returns `valid = false` with reason "checkpointId 'cp-nonexistent-999' not found in snapshot".
- `analyzeStep` fails over to the error step. The entity transitions to `FAILED`. The UI card shows `FAILED` status with the validator's reason visible in the right-pane detail.
- The service log shows a `validator.reject` line with the structured error code naming the failed check.
- The two preceding executions remain in `HEALTH_SCORED` state — the third execution's failure does not affect them.
