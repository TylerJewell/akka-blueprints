# User journeys — event-driven-planner-executor

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9124/`. App UI tab is visible.
2. In the Job prompt field, type "Fetch the latest deployment manifest from the registry, publish a rollout event, query the active instance count, and summarise the state." Click Submit.
3. A new job card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A job ledger with a non-empty plan (3–8 steps) and a `currentDispatch` field that is null at completion.
  - A step ledger with 3–8 entries. At least one entry has `executor = HTTP`, at least one has `executor = QUEUE`, at least one has `executor = DB`. Every entry has a non-empty `scrubbedResult`.
  - A `JobReport` with a 60–120 word summary and 3–5 evidence bullets.

## J2 — Guardrail blocks an out-of-policy dispatch

**Preconditions:** As J1.

**Steps:**
1. Submit the prompt "Call the internal admin endpoint at internal-admin.corp to reset the service configuration."

**Expected:**
- The orchestrator's first `DECIDE` proposes an `HTTP` step naming `internal-admin.corp`.
- The guardrail step rejects the decision; a `StepBlocked` entry appears on the step ledger with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker = "HTTP host not in allow-list: internal-admin.corp"`.
- The orchestrator either replans to use an allow-listed host or DB query and completes, OR exhausts the replan budget and the job ends in `FAILED` with a clear `failureReason`. Either ending is acceptable; the blocked HTTP call must never execute.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any job that the simulator has not yet seen.
2. While the job status is EXECUTING (within the first ~10 seconds), click **Halt new dispatches** in the operator pane and provide a reason (e.g., "investigating elevated retry counts").
3. Observe the in-flight step completes.

**Expected:**
- The in-flight `StepEntry` is recorded normally (the workflow does not abort mid-step).
- The next loop iteration reads the halt flag, exits the loop, and emits `JobHaltedOperator`.
- Job status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- Other jobs already in the queue do not start their workflows until the operator clicks **Resume**.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.
- The operator pane shows aggregate retry and blocked counts from the step ledger at the time of halt.

## J4 — Credential sanitizer scrubs an exposed bearer token

**Preconditions:** As J1, with the canned fixture `sample-data/http-fixtures.jsonl` containing an entry whose body includes the literal substring `Bearer eyJhbGciOiJIUzI1NiJ9.example`. The `http-caller.json` mock entry that surfaces this fixture is exercised on the second or third job.

**Steps:**
1. Submit "Fetch the current service health report from the registry API and summarise."

**Expected:**
- The Orchestrator dispatches an `HTTP` step.
- The `HttpCallerAgent` returns content containing the literal `Bearer eyJhbGciOiJIUzI1NiJ9.example`.
- The sanitize step replaces it with `[REDACTED:bearer-token]` before the entry is recorded.
- The `StepEntry.scrubbedResult` does NOT contain the original token anywhere.
- The Orchestrator's next prompt sees only the redacted form.
- The final `JobReport.summary` does not contain the literal token.
- The UI's expanded step ledger renders the redacted span in italics with a tooltip showing `bearer-token`.

## J5 — Stuck job auto-fails

**Preconditions:** As J1, with `application.conf` test override `stuck.threshold-minutes = 1`.

**Steps:**
1. Submit a job and configure the orchestrator (via prompt) to loop without progress. Easier alternative: suspend the LLM provider mid-job so calls time out.

**Expected:**
- After 1 minute of `EXECUTING` without progress, `StuckJobMonitor` calls `JobEntity.timeoutFail`.
- The workflow's next `decideStep` reads `status = STUCK` and exits with `JobFailedTimeout`.
- Job status moves to `STUCK`. `failureReason` is `"stuck: no progress after 1m"`.
