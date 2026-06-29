# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J4 pass.

## J1 — Submit a writing task and see it complete

**Preconditions:** service running on `http://localhost:9624/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a task description that clearly requires drafting (e.g., "Write a short summary of agile sprint planning for a new team member") and Submit.
2. Observe the new task row via SSE.

**Expected:** the task progresses `QUEUED → PROCESSING → COMPLETED` within ~60 s. The expanded row shows a `DraftOutput` (60–120 words of drafted text) and an `AssembledResult` whose `toolsUsed` is `["writer_tool"]`. The `DataOutput` field is null — the supervisor called only the writing tool.

## J2 — Submit a task that needs both tools

**Preconditions:** service running; a model provider configured.

**Steps:**
1. Submit a task description that requires both writing and entity extraction (e.g., "Extract the key people and organisations from this memo about the merger, and draft a one-paragraph summary.").
2. Observe the task row.

**Expected:** the task enters `COMPLETED`. The expanded row shows both a `DraftOutput` and a `DataOutput` (3–5 entities). The `AssembledResult.toolsUsed` lists both `"writer_tool"` and `"data_tool"`. The `answer` references both outputs.

## J3 — Submit an unroutable task

**Preconditions:** service running; a model provider configured.

**Steps:**
1. Submit a task description that neither tool can handle (e.g., "Translate the following paragraph into French." or "Run a SQL query against the orders database.").
2. Observe the task row.

**Expected:** the task enters `REJECTED`. The expanded row shows a `rejectionReason` explaining why neither tool applies. No `DraftOutput` or `DataOutput` is set. The row's status pill shows "REJECTED" in a distinct colour.

## J4 — Quality score appears beside a completed task

**Preconditions:** at least one `COMPLETED` task without a `qualityScore`.

**Steps:**
1. Wait for `QualitySampler` to run (every 5 minutes), or trigger it manually.
2. Observe the task row in the live list.

**Expected:** the task row gains a `qualityScore` (1–5) and a `qualityRationale`; the App UI row shows the score beside the status pill. Task delivery was not blocked by the scoring (non-blocking self-transition on `COMPLETED`).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `TaskSimulator` drips a task description from `sample-tasks.jsonl` every 60 s; each becomes a task record that flows through the full pipeline. The App UI is non-empty on first load.

## J6 — Supervise step timeout fails the task gracefully

**Preconditions:** `superviseStep` timeout set to 1 s (test override).

**Steps:**
1. Submit any task.
2. Watch the task row.

**Expected:** the supervise step times out before the supervisor returns. The workflow routes to `failStep`; the task enters `FAILED` with a `rejectionReason` noting the timeout. No infinite retry; the record is terminal.
