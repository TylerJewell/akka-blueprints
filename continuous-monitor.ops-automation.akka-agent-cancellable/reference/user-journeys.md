# User journeys — activity-interrupt-cancellation

## J1 — Task arrives and starts running

**Preconditions:** Service running on port 9855; valid model-provider API key set (or mock LLM enabled); `TaskPoller` enabled.

**Steps:**
1. Open `http://localhost:9855/` → App UI tab.
2. Wait up to 20 s for the first simulated task.

**Expected:**
- Task card appears with status QUEUED, then transitions to RUNNING within 1 s.
- The step log timeline grows: each completed step appends a row with its summary and an optional artefact chip.
- The Cancel button is active (not greyed) while the task is RUNNING.

## J2 — Operator cancels a running task

**Preconditions:** A task in RUNNING status.

**Steps:**
1. Click Cancel on the task card.
2. In the confirmation dialog, enter a reason (e.g., "Wrong environment targeted").
3. Confirm.

**Expected:**
- Status transitions to CANCELLING immediately (SSE event arrives < 1 s).
- The running step log entry shows "Cancelled at operator request" as the final step.
- Status transitions to CANCELLED after the cleanup plan is produced (typically within 30 s).
- The cleanup plan block appears in the detail view, listing the remediation steps.
- `finishedAt` is populated in the entity record.

## J3 — Task runs to completion without cancellation

**Preconditions:** A task in RUNNING status; no operator action taken.

**Steps:**
1. Observe the task card while the agent works through all steps.

**Expected:**
- Status reaches COMPLETED after the agent's final step returns `terminal=true`.
- No cleanup plan block appears in the detail view.
- The Cancel button becomes inactive once COMPLETED.

## J4 — Stale reaper auto-cancels an overdue task

**Preconditions:** Service started with `STALE_TIMEOUT_SECONDS=30` (reduced for the test). A task is in RUNNING status.

**Steps:**
1. Wait 30 s without clicking Cancel.

**Expected:**
- Status transitions to CANCELLING, then CANCELLED, automatically.
- `cancellationRequest.requestedBy` is `"reaper"`.
- A cleanup plan is recorded (same path as operator cancel).
- The task card shows a "Timed out" badge distinguishing it from operator-cancelled tasks.

## J5 — Cancelled task's step log is intact

**Preconditions:** A task that has been cancelled (J2 or J4) with at least two completed steps.

**Steps:**
1. Select the cancelled task in the list.
2. Inspect the step log in the detail pane.

**Expected:**
- All steps that completed before the cancellation are visible with their summaries and artefact chips.
- The final step is the cancellation step.
- The cleanup plan references the artefacts by name when they appear in `partialArtifact` fields of earlier steps.
