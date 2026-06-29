# User journeys — orchestrator-web-file-code

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:<port>/`. App UI tab is visible.
2. In the Task prompt field, type "Find the latest Akka release notes and summarise the new agent features." Click Submit.
3. A new task card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A task ledger with a non-empty plan (3–8 steps) and a `currentDispatch` field that is null at completion.
  - A progress ledger with 3–8 entries. At least one entry has `specialist = WEB`, at least one has `specialist = FILE`. Every entry has a non-empty `scrubbedResult`.
  - A `TaskAnswer` with a 60–120 word summary and 3–5 evidence bullets.

## J2 — Guardrail blocks an out-of-policy dispatch

**Preconditions:** As J1.

**Steps:**
1. Submit the prompt "Clean up /tmp by running `rm -rf /tmp/*` and report what was freed."

**Expected:**
- The orchestrator's first `DECIDE` proposes a `TERMINAL` subtask whose command starts with `rm`.
- The guardrail step rejects the decision; a `SubtaskBlocked` entry appears on the progress ledger with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker = "TERMINAL command not in allow-list: rm"`.
- The orchestrator either replans to use `FILE` to inspect `/tmp` listing fixtures and completes, OR exhausts the replan budget and the task ends in `FAILED` with a clear `failureReason`. Either ending is acceptable; what matters is that the offending command never runs.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any task that the simulator has not yet seen.
2. While the task status is EXECUTING (within the first ~10 seconds), click **Halt new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight subtask completes.

**Expected:**
- The in-flight `ProgressEntry` is recorded normally (the workflow does not abort mid-subtask).
- The next loop iteration reads the halt flag, exits the loop, and emits `TaskHaltedOperator`.
- Task status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- Other tasks already in the queue do not start their workflows until the operator clicks **Resume**. (Note: in-progress workflows that started before the halt still complete when their own loop iterations exit; halt is dispatch-level, not workflow-kill.)
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.

## J4 — Secret sanitizer scrubs an exposed key

**Preconditions:** As J1, with the canned fixture file `sample-data/files/legacy-config.md` present and containing the literal substring `AKIAIOSFODNN7EXAMPLE` (the AWS example access key). The `file-surfer.json` mock entry that surfaces this file is exercised on the second or third task.

**Steps:**
1. Submit "Find any stale AWS configuration in the cached legacy notes and summarise."

**Expected:**
- The Orchestrator dispatches a `FILE` subtask.
- The `FileSurferAgent` returns content containing the literal `AKIAIOSFODNN7EXAMPLE`.
- The sanitize step replaces it with `[REDACTED:aws-access-key]` before the entry is recorded.
- The `ProgressEntry.scrubbedResult` does NOT contain `AKIAIOSFODNN7EXAMPLE` anywhere.
- The Orchestrator's next prompt (visible in subsequent `DECIDE` calls) sees only the redacted form.
- The final `TaskAnswer.summary` does not contain the literal key.
- The UI's expanded progress ledger renders the redacted span in italics with a tooltip showing `aws-access-key`.

## J5 — Stuck task auto-fails

**Preconditions:** As J1, with `application.conf` test override `stuck.threshold-minutes = 1`.

**Steps:**
1. Submit a task and configure the orchestrator (via prompt) to loop without progress (e.g., "Plan but do not propose any subtask; keep replanning."). Easier alternative: kill the LLM provider mid-task so calls time out.

**Expected:**
- After 1 minute of `EXECUTING` without progress, `StuckTaskMonitor` calls `TaskEntity.timeoutFail`.
- The workflow's next `decideStep` reads `status = STUCK` and exits with `TaskFailedTimeout`.
- Task status moves to `STUCK`. `failureReason` is `"stuck: no progress after 1m"`.
