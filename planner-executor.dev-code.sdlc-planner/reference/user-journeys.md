# User journeys — sdlc-task-planner

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:9558/`. App UI tab is visible.
2. In the SDLC request field, type "Analyse, design, implement, and review a feature that adds rate-limiting to a REST endpoint." Click Submit.
3. A new plan card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A task ledger with a non-empty plan (3–8 steps) and a `currentDispatch` field that is null at completion.
  - A progress ledger with 3–8 entries. At least one entry has `specialist = ANALYST`, at least one has `specialist = CODER`. Every entry has a non-empty `scrubbedResult`.
  - A `PlanDeliverable` with a 60–120 word summary and 3–5 artefact bullets.

## J2 — Guardrail blocks an out-of-scope code write

**Preconditions:** As J1.

**Steps:**
1. Submit the prompt "Write a cron job script to `/etc/cron.d/cleanup` that removes build artefacts nightly."

**Expected:**
- The planner's first `DECIDE` proposes a `CODER` sub-task referencing `/etc/cron.d/`.
- The guardrail step rejects the decision; a `SubtaskBlocked` entry appears on the progress ledger with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker = "CODER path outside /workspace/: /etc/cron.d/cleanup"`.
- The planner either replans (e.g., moves the script to `/workspace/scripts/cleanup`) and completes, OR exhausts the replan budget and the plan ends in `FAILED` with a clear `failureReason`. Either ending is acceptable; what matters is that the write outside `/workspace/` never proceeds.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any SDLC request that the simulator has not yet seen.
2. While the plan status is EXECUTING (within the first ~10 seconds), click **Halt new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight sub-task completes.

**Expected:**
- The in-flight `ProgressEntry` is recorded normally (the workflow does not abort mid sub-task).
- The next loop iteration reads the halt flag, exits the loop, and emits `PlanHaltedOperator`.
- Plan status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- Plans already in the queue do not start their workflows until the operator clicks **Resume**.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.

## J4 — Secret sanitizer scrubs an exposed key

**Preconditions:** As J1, with the canned fixture file `sample-data/code-snippets/legacy-config.yaml` present and containing the literal substring `AKIAIOSFODNN7EXAMPLE`. The `reviewer.json` mock entry that surfaces this file is exercised on the second or third plan.

**Steps:**
1. Submit "Review the legacy configuration file for security issues and summarise findings."

**Expected:**
- The Planner dispatches a `REVIEWER` sub-task.
- The `ReviewerAgent` returns content containing the literal `AKIAIOSFODNN7EXAMPLE`.
- The sanitize step replaces it with `[REDACTED:aws-access-key]` before the entry is recorded.
- The `ProgressEntry.scrubbedResult` does NOT contain `AKIAIOSFODNN7EXAMPLE` anywhere.
- The Planner's next prompt sees only the redacted form.
- The final `PlanDeliverable.summary` does not contain the literal key.
- The UI's expanded progress ledger renders the redacted span in italics with a tooltip showing `aws-access-key`.

## J5 — Stuck plan auto-fails

**Preconditions:** As J1, with `application.conf` test override `stuck.threshold-minutes = 1`.

**Steps:**
1. Submit a plan and configure the planner (via prompt) to loop without progress (e.g., "Keep re-analysing without dispatching a Coder sub-task."). Easier alternative: kill the LLM provider mid-plan so calls time out.

**Expected:**
- After 1 minute of `EXECUTING` without progress, `StalePlanMonitor` calls `PlanEntity.timeoutFail`.
- The workflow's next `decideStep` reads `status = STUCK` and exits with `PlanFailedTimeout`.
- Plan status moves to `STUCK`. `failureReason` is `"stuck: no progress after 1m"`.
