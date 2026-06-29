# User journeys — sandboxed-analyst-agent

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on port 9822; valid model-provider API key set (or `model-provider = mock`); E2B API key set (or mock-execution mode active).

**Steps:**
1. Open `http://localhost:9822/`. App UI tab is visible.
2. In the Dataset name field, type "q1-2026-inbound-calls.csv". Click Upload.
3. A new job card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - An analysis ledger with a non-empty plan (3–8 steps) and a `currentStep` field that is null at completion.
  - A progress ledger with 3–8 entries. At least one entry has `scriptKind = LOAD_INSPECT`, at least one has `scriptKind = AGGREGATE`. Every entry has a non-empty `scrubbedOutput`.
  - An `AnalysisReport` with a 60–120 word summary, at least 2 `AnalysisFinding` records, and a non-empty `charts` list.

## J2 — Guardrail blocks an out-of-policy script

**Preconditions:** As J1.

**Steps:**
1. Submit a dataset with a description that would cause the analyst to propose a script containing `import requests` to fetch supplementary data from an external URL.

**Expected:**
- The analyst's first `DECIDE` proposes a `LOAD_INSPECT` or other step whose script contains `import requests`.
- The guardrail step rejects the script; a `StepBlocked` entry appears on the progress ledger with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker` naming the forbidden import.
- The analyst either replans to use only allowed libraries (`pandas`, `numpy`, `re`) and completes, OR exhausts the replan budget and the job ends in `FAILED` with a clear `failureReason`. Either ending is acceptable; what matters is that the offending import never reaches the sandbox.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any dataset the simulator has not yet seen.
2. While the job status is EXECUTING (within the first ~10 seconds), click **Halt new steps** in the operator pane and provide a reason.
3. Observe the in-flight sandbox execution completes.

**Expected:**
- The in-flight `ProgressEntry` is recorded normally (the workflow does not abort mid-execution).
- The next loop iteration reads the halt flag, exits the loop, and emits `JobHaltedOperator`.
- Job status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- Other jobs already in the queue do not start their workflows until the operator clicks **Resume**.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.

## J4 — PII sanitizer scrubs phone numbers

**Preconditions:** As J1, with the canned fixture file `sample-data/calls/contacts-q1.csv` present and containing a column of E.164 phone numbers (e.g., `+12025551234`). The `sandbox-executor.json` mock entry that surfaces this file is exercised on the second or third job.

**Steps:**
1. Submit "Identify all customer contacts from the Q1 inbound call records."

**Expected:**
- The AnalystAgent dispatches a `LOAD_INSPECT` step that reads the contacts CSV.
- The `SandboxExecutorAgent` returns stdout containing the literal `+12025551234`.
- The sanitize step replaces it with `[REDACTED:phone]` before the entry is recorded.
- The `ProgressEntry.scrubbedOutput` does NOT contain `+12025551234` or any other unredacted phone number.
- The AnalystAgent's next prompt (visible in subsequent `DECIDE` calls) sees only the redacted form.
- The final `AnalysisReport.summary` does not contain any literal phone number.
- The UI's expanded progress ledger renders the redacted span in italics with a tooltip showing `phone`.

## J5 — Stale job auto-fails

**Preconditions:** As J1, with `application.conf` test override `stale.threshold-minutes = 1`.

**Steps:**
1. Submit a dataset and configure the analyst (via a prompt that causes it to loop without progress, e.g., "Plan but propose no scripts; keep replanning."). Easier alternative: halt the E2B connection mid-job so all calls time out.

**Expected:**
- After 1 minute of `EXECUTING` without progress, `StaleJobMonitor` calls `AnalysisJobEntity.timeoutFail`.
- The workflow's next `decideStep` reads `status = STUCK` and exits with `JobFailedTimeout`.
- Job status moves to `STUCK`. `failureReason` is `"stuck: no progress after 1m"`.
