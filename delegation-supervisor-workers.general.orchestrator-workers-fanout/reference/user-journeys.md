# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a task and watch parallel execution

**Preconditions:** service running on `http://localhost:9695/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a task description and Submit.
2. Observe the new task row via SSE.

**Expected:** the task progresses `PLANNING → IN_PROGRESS → COMPLETED` within ~90 s. The expanded row shows an `ExecutionPlan` (two sub-tasks), a `WriterOutput` (content + sections), an `AnalyzerOutput` (evaluation + findings), and a composite summary. Writer and analyzer outputs arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the task

**Preconditions:** `WriterWorker` step timeout set to 1 s (test override).

**Steps:**
1. Submit a task.
2. Watch the task row.

**Expected:** the `writeStep` times out, the workflow routes to `degradeStep`, and the Orchestrator synthesises from the AnalyzerWorker output alone. The task enters `DEGRADED`; the summary notes the missing side. No infinite retry.

## J3 — Plan guardrail blocks a malformed plan

**Preconditions:** Orchestrator configured to return a plan with an unknown `workerRole` value (test fixture).

**Steps:**
1. Submit the fixture task.
2. Watch the task row.

**Expected:** `planGuardrailStep` flags the invalid plan; the workflow calls `block`; the task enters `BLOCKED` with a `failureReason`. Neither `WriterWorker` nor `AnalyzerWorker` is invoked. The task never reaches `IN_PROGRESS`.

## J4 — Eval score appears beside a completed task

**Preconditions:** at least one `COMPLETED` task without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it.
2. Observe the task row.

**Expected:** the task gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `TaskSimulator` drips a task from `task-requests.jsonl` every 60 s; each becomes a task request that flows through the full pipeline. The App UI is non-empty on first load.
