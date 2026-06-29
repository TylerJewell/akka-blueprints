# User journeys — autonomous-agent-pattern

## J1 — Submit an issue and receive a patch

**Preconditions:** Service running on `http://localhost:9758/`. A valid `GITHUB_TOKEN` set in the environment (or the mock LLM selected at scaffold time). A seeded GitHub issue URL loaded.

**Steps:**
1. Open `http://localhost:9758/` → App UI tab.
2. Click **Load seeded example** to populate the issue URL field with the seeded null-pointer bug issue.
3. Click **Submit**.
4. Watch the card reach `CHECKPOINT_PENDING`. The right pane shows the Plan panel with `filesToChange` and `approach`.
5. Click **Approve**.
6. Watch the card move through `EXECUTING`, then reach `PATCH_READY`.

**Expected:**
- The card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `ISSUE_FETCHED` within 2 s (mock mode) or up to 10 s (real GitHub API).
- The card transitions to `PLAN_READY` once the plan agent returns (within 30 s real, 2 s mock).
- The card reaches `CHECKPOINT_PENDING` and the Checkpoint panel shows the plan with Approve / Reject buttons.
- After Approve, the card transitions to `EXECUTING`. The tool-call trace panel updates with each tool call as it lands via SSE.
- Within 60 s (real) or 5 s (mock), the card reaches `PATCH_READY`. The patch diff panel shows a non-empty unified diff, a confidence score ≥ 0.7, and a tool trace with at least one `readFile` and one `writeFile` call, all with status `SUCCEEDED`.

## J2 — Before-tool-call guardrail blocks a destructive shell command

**Preconditions:** Service running. Mock LLM mode (the mock `RESOLVE_ISSUE` entry includes a `runShell` call with `command = "rm -rf /workspace"`).

**Steps:**
1. Submit the seeded null-pointer bug issue (J1 steps 2–5).
2. Watch the tool-call trace panel in real time as `EXECUTING`.
3. Look for the blocked tool-call row.

**Expected:**
- A tool-call row appears with `toolName = runShell`, `status = BLOCKED`, and `blockReason` = `"shell-danger-pattern: 'rm -rf' matches disallowed pattern"`.
- The row renders in red italic in the UI.
- The entity records a `ToolCallExecuted` event with `status = BLOCKED` for that call — no file system operation occurred.
- The agent receives the rejection reason and proposes an alternative tool call in the next step. The task eventually reaches `PATCH_READY` (the mock's next iteration produces a valid outcome).
- The service log contains one `guardrail.block` entry with the structured reason for the blocked call.

## J3 — Operator halts a running agent loop

**Preconditions:** Service running. Any model provider. A task has been submitted and approved (J1 steps 1–5). The task is in `EXECUTING` status.

**Steps:**
1. While the task card shows `EXECUTING`, click the red **Halt** button in the right pane's header.

**Expected:**
- `POST /api/tasks/{id}/halt` returns `200`.
- The entity records `HaltRequested` (setting `haltRequested = true`) and then `TaskHalted` within one iteration timeout (≤ 60 s real, ≤ 5 s mock).
- The card transitions to `HALTED` with status pill in orange.
- The tool-call trace panel shows the partial trace up to the point the halt was detected. No further tool calls appear after the halt event timestamp.
- `GET /api/tasks/{id}` returns `"status": "HALTED"` and `"finishedAt"` set to the halt timestamp. The `outcome` field is `null`.

## J4 — Human checkpoint rejects the plan; no writes occur

**Preconditions:** Service running. Any model provider. A task is in `CHECKPOINT_PENDING` status.

**Steps:**
1. With a task card at `CHECKPOINT_PENDING`, read the Plan panel.
2. Click the red **Reject** button.

**Expected:**
- `POST /api/tasks/{id}/reject` returns `200`.
- The entity records `CheckpointRejected` and then `TaskFailed{reason: "checkpoint-rejected"}`.
- The card transitions to `FAILED` with status pill in red.
- The tool-call trace panel is empty — no `ToolCallExecuted` events were emitted after the reject (no file writes occurred).
- `GET /api/tasks/{id}` returns `"status": "FAILED"` and `"outcome": null`.

## J5 — RuntimeMonitor detects a rate spike and raises an alert

**Preconditions:** Service running. Mock LLM mode. A mock `RESOLVE_ISSUE` entry that produces 11 `ToolCall` events within a single task (exceeding the 10-per-minute threshold).

**Steps:**
1. Submit the seeded date-range-filter issue (J1 steps 2–5, using the second seeded issue).
2. Watch the alert chip strip on the task detail panel.

**Expected:**
- After the 11th `ToolCallExecuted` event lands on the entity, `RuntimeMonitor` emits a `MonitorAlert{kind: RATE_SPIKE, detail: "11 tool calls in 60s window for task <id>"}`.
- The entity records `MonitorAlertRaised`.
- An orange alert chip with label `RATE_SPIKE` appears in the alert strip on the UI within 1 s of the event.
- The agent loop continues — the alert does not halt execution.
- `GET /api/tasks/{id}` returns `"alerts": [{ "alertId": "...", "kind": "RATE_SPIKE", ... }]`.

## J6 — Seeded issue with multi-file plan triggers broad-plan warning

**Preconditions:** Service running. Mock LLM mode using the mock PLAN_ISSUE entry that returns more than 10 files in `filesToChange`.

**Steps:**
1. Submit the seeded missing-validation issue.
2. Wait for `CHECKPOINT_PENDING`.
3. Read the Plan panel.

**Expected:**
- The Plan panel shows a `filesToChange` chip list with more than 10 entries.
- The UI renders a yellow warning banner: "This plan touches many files — verify scope before approving."
- The Approve and Reject buttons are both enabled; the warning is informational, not blocking.
- If the user approves, the workflow proceeds normally. If the mock execution entry for this task returns `OutcomeStatus.PARTIAL`, the outcome badge shows `PARTIAL` in yellow and the confidence score bar is below 0.5.
