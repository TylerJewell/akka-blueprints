# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a task and watch parallel worker execution

**Preconditions:** service running on `http://localhost:9925/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a task description (e.g., "Compare global renewable energy adoption rates and show trends by region over the last 5 years.") and Submit.
2. Observe the new task row via SSE.

**Expected:** the task progresses `ROUTING → IN_PROGRESS → COMPLETED` within ~60 s. The expanded row shows a `ResearchOutput` (3–5 findings), a `ChartData` payload (2–3 series), and a synthesised summary. Research and chart data arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the task

**Preconditions:** `ResearchWorker` step timeout set to 1 s (test override).

**Steps:**
1. Submit a task description.
2. Watch the task row.

**Expected:** the `researchStep` times out, the workflow routes to `degradeStep`, and the supervisor synthesises from the ChartWorker output alone. The task enters `DEGRADED`; the summary notes the missing research side. No infinite retry.

## J3 — Before-tool-call guardrail blocks a disallowed tool invocation

**Preconditions:** supervisor prompt or test fixture configured to attempt a tool call outside the permitted list (e.g., `FILE_READ`).

**Steps:**
1. Submit a task that triggers the disallowed call.
2. Watch the task row.

**Expected:** `guardrailStep` intercepts the call before it executes; the workflow calls `blockTask`; the task enters `BLOCKED` with a `failureReason` identifying the disallowed tool. The result is never surfaced as COMPLETED in the App UI.

## J4 — Eval score appears beside a completed task

**Preconditions:** at least one `COMPLETED` task without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it directly.
2. Observe the task row.

**Expected:** the task gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery was not blocked by the evaluation (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `TaskSimulator` drips a task description from `sample-tasks.jsonl` every 60 s; each becomes a task that flows through the full pipeline. The App UI is non-empty on first load.
