# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a multi-file task and watch parallel editing

**Preconditions:** service running on `http://localhost:9579/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a description such as "Add null-guard to all public methods" and two target file paths. Submit.
2. Observe the new task row via SSE.

**Expected:** the task progresses `PLANNING → IN_PROGRESS → COMPLETED` within ~90 s. The expanded row shows a `DecompositionPlan` with one `FileInstruction` per file, an `EditedFile` per file with a diff summary, a `ReviewVerdict` per file, and a changeset summary. Edit outputs arrive close together because the workers ran in parallel.

## J2 — Before-tool-call guardrail blocks an out-of-scope write

**Preconditions:** service running; `FileEditor` configured to attempt a write to a path not in the declared `targetFiles` (test fixture).

**Steps:**
1. Submit a task with a specific `targetFiles` list.
2. Observe the task row.

**Expected:** the before-tool-call guardrail intercepts the out-of-scope `applyEdit` call and rejects it. The task transitions to `BLOCKED` with a `failureReason` identifying the rejected path. No further worker steps run. The task is never marked `COMPLETED`.

## J3 — Worker timeout degrades the task

**Preconditions:** `FileEditor` step timeout set to 1 s (test override).

**Steps:**
1. Submit a task with two target files.
2. Watch the task row.

**Expected:** at least one `editStep` times out; the workflow routes to `degradeStep` and the Orchestrator synthesises from whichever files completed. The task enters `DEGRADED`; the changeset summary notes the missing file. No infinite retry.

## J4 — Quality score appears beside a completed task

**Preconditions:** at least one `COMPLETED` task without a `qualityScore`.

**Steps:**
1. Wait for `QualitySampler` to run (every 5 minutes), or trigger it manually.
2. Observe the task row.

**Expected:** the task gains a `qualityScore` (1–5) and a `qualityRationale`; the App UI row shows the score. Task delivery was never delayed by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `TaskSimulator` drips an edit task from `edit-tasks.jsonl` every 60 s; each becomes a task that flows through the full pipeline. The App UI is non-empty on first load.

## J6 — Review verdict marks a changeset as needing review

**Preconditions:** `EditReviewer` returns `approved = false` for one file (test fixture).

**Steps:**
1. Submit a task.
2. Wait for synthesis.

**Expected:** the Orchestrator sets `qualityVerdict = "needs-review"` in the `Changeset`. The App UI shows the task as `COMPLETED` (delivery is not blocked) but the expanded row displays the `needs-review` verdict and the reviewer's feedback. The `QualitySampler` reflects the reduced quality in its score.
