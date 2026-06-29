# User journeys — workflow-orchestration

## J1 — Submit a workflow and get a run result

**Preconditions:** Service running on declared port (`http://localhost:9807/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded workflow `database-migration-v3` has a matching `src/main/resources/sample-data/workflows/database-migration-v3.json` file.

**Steps:**
1. Open `http://localhost:9807/` → App UI tab.
2. From the **Pick a seeded workflow** dropdown, pick `database-migration-v3`.
3. Click **Run workflow**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `VALIDATING` within 1 s more.
- Within ~20 s the card reaches `VALIDATED`. The right pane shows the Validation-report table with ≥ 3 rows; each row has a non-empty stepId and status of `OK` or `WARN`.
- Within ~20 s more the card reaches `EXECUTED`. The right pane shows ≥ 3 step outcomes; every `outcome.stepId` matches a `stepId` from the validation table; every `outcome.status` is `SUCCEEDED`, `FAILED`, or `SKIPPED`.
- Within ~20 s more the card reaches `NOTIFIED`, then `EVALUATED` within 1 s of that. The right pane shows a `RunResult` with `stepOutcomes.length == validations.length`, a summary with correct totals, and a notification receipt with a non-empty `messageId`. The eval score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Stage-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `validate-steps.json` includes one entry whose `tool_calls` array starts with a NOTIFY-stage tool (`dispatchNotification`) — this is the deliberately stage-violating entry.

**Steps:**
1. Submit any seeded workflow three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/runs/sse`).

**Expected:**
- On the third submission's `validateStage`, the agent's first iteration calls `dispatchNotification`. `StageGuardrail` rejects it; a `StageGuardrailRejected{stage: "VALIDATE", tool: "dispatchNotification", reason: "stage-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `NotifyTools.dispatchNotification` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal validate sequence (`checkStepDependencies` + `verifyStepPreconditions`). The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Phantom step flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `notify-completion.json` includes one entry whose paired `ExecutionResult` contains a `stepId` absent from the workflow declaration's `ValidationReport`.

**Steps:**
1. Submit any seeded workflow six times. (The phantom-step entry is selected once in every six runs by the mock's `seedFor(runId)` modulo.)
2. Watch the live list until a run card's border highlights red.

**Expected:**
- The flagged run lands well-formed (the guardrail only checks stage order, not step provenance).
- The eval score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads: *"Phantom step check failed: outcome for stepId 'cleanup-tmp' does not appear in the ValidationReport validations."*
- The card's border highlights red. The operator knows to inspect this run before triggering follow-on actions.
- The other five runs in the set scored ≥ 4.

## J4 — Per-stage tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded workflow.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the runId.

**Expected:**
- The VALIDATE task's log entries show only `checkStepDependencies` and `verifyStepPreconditions` calls.
- The EXECUTE task's log entries show only `runStep` and `recordStepOutcome` calls.
- The NOTIFY task's log entries show only `buildRunSummary` and `dispatchNotification` calls.
- No cross-stage calls appear. (If the mock-LLM violation path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.)
- The order of tasks in the log is VALIDATE → EXECUTE → NOTIFY. Each task's first log line is preceded by a `workflow.step.start` line for the matching stage.

## J5 — Step count parity

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded workflow `service-deployment-canary` (whose mock-paired `ValidationReport` carries 5 steps).
2. Wait for `EVALUATED`.

**Expected:**
- The recorded `RunResult.stepOutcomes.length` equals the recorded `ExecutionResult.outcomes.length` equals `ValidationReport.validations.length` (all 5).
- Every `outcome.stepId` matches one `validation.stepId` from the report, one-to-one.
- If the agent's first iteration produced 4 outcomes instead of 5 (silent collapse), the on-completion evaluator would have flagged it with a "count parity" failure and a score of 4. The presence of a score of 5 with rationale "all checks passed" confirms parity held.

## J6 — Unknown workflow id

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/workflows/<unknown-id>.json` exists.

**Steps:**
1. In the App UI, type a workflow id that has no matching sample file (e.g. `hypothetical-cleanup-job`).
2. Click **Run workflow**.

**Expected:**
- `ValidateTools.checkStepDependencies` returns an empty `List<StepValidation>` (no steps found in the catalog).
- The agent's VALIDATE task returns a `ValidationReport` with `validations = []` and `allPassed = true` (vacuously — no steps to fail).
- The pipeline advances to `executeStage`. The EXECUTE task returns an `ExecutionResult` with `outcomes = []`.
- The pipeline advances to `notifyStage`. The NOTIFY task returns a `RunResult` with `stepOutcomes = []` and a summary of `totalSteps = 0`.
- The eval score chip shows 1 (count parity is satisfied at 0 == 0, but no other rule yields a point). The rationale notes "no steps to evaluate."
- The pipeline completes; nothing crashes; the empty run is honestly empty.
