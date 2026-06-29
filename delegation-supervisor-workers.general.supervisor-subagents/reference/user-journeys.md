# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a task and watch parallel subagent execution

**Preconditions:** service running on `http://localhost:9988/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a task description and Submit.
2. Observe the new task row via SSE.

**Expected:** the task progresses `RECEIVED → ROUTING → IN_PROGRESS → COMPLETED` within ~90 s. The expanded row shows a `RoutingPlan` with dataQuery and summaryPrompt, a `DataBundle` (3–5 items), a `SummaryOutput` (3–5 key points), and an assembled narrative. The data bundle and summary arrive close together because both subagents ran in parallel.

## J2 — Subagent timeout degrades the task

**Preconditions:** `DataSubagent` step timeout set to 1 s (test override).

**Steps:**
1. Submit a task description.
2. Watch the task row.

**Expected:** `fetchDataStep` times out, the workflow routes to `degradeStep`, and the TaskSupervisor assembles from the SummarySubagent output alone. The task enters `DEGRADED`; the assembled narrative notes the missing data side in one sentence. No infinite retry.

## J3 — Routing guardrail blocks a disallowed task

**Preconditions:** TaskSupervisor returns a RoutingPlan that the guardrail is configured to reject (test fixture with a disallowed dataQuery keyword).

**Steps:**
1. Submit the fixture task description.
2. Watch the task row.

**Expected:** `guardrailStep` rejects the routing plan; the workflow calls `blockTask`; the task enters `BLOCKED` with a `failureReason`. Neither DataSubagent nor SummarySubagent was invoked. The task never appears as COMPLETED or IN_PROGRESS in the App UI.

## J4 — Routing eval score appears beside a completed task

**Preconditions:** at least one `COMPLETED` task without an `evalScore`.

**Steps:**
1. Wait for `RoutingEvalSampler` to run (every 5 minutes), or trigger it.
2. Observe the task row.

**Expected:** the task gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Task delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `TaskSimulator` drips a task description from `sample-tasks.jsonl` every 60 s; each becomes a task that flows through the full pipeline. The App UI is non-empty on first load.
