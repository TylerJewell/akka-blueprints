# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J4 pass.

## J1 — Submit a job and watch parallel processing

**Preconditions:** service running on `http://localhost:9386/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a job payload and Submit.
2. Observe the new job row via SSE.

**Expected:** the job progresses `QUEUED → PROCESSING → CONSOLIDATED` within ~60 s. The expanded row shows a `StructuralOutput` (2–5 elements), a `ContextualOutput` (3–6 tags with enriched context), and a consolidated summary. Structural and contextual outputs arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the job

**Preconditions:** `SubtaskWorkerA` step timeout set to 1 s (test override).

**Steps:**
1. Submit a job payload.
2. Watch the job.

**Expected:** the `structuralStep` times out, the workflow routes to `degradeStep`, and the coordinator consolidates from the contextual output alone. The job enters `DEGRADED`; the summary notes the missing structural side. No infinite retry.

## J3 — Validation rejects a malformed result

**Preconditions:** coordinator returns a result with an empty summary (test fixture).

**Steps:**
1. Submit the fixture payload.
2. Watch the job.

**Expected:** `validateStep` rejects the consolidated content; the workflow calls `reject`; the job enters `REJECTED` with a `failureReason`. The job's status pill shows REJECTED in the App UI list.

## J4 — Quality score appears beside a consolidated job

**Preconditions:** at least one `CONSOLIDATED` job without a `qualityScore`.

**Steps:**
1. Wait for `QualitySampler` to run (every 5 minutes), or trigger it directly.
2. Observe the job row in the App UI.

**Expected:** the job gains a `qualityScore` (1–5) and a `qualityRationale`; the App UI row shows the score. Delivery was never blocked by the quality check (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `JobSimulator` drips a payload from `sample-jobs.jsonl` every 60 s; each becomes a job that flows through the full pipeline. The App UI is non-empty on first load.

## J6 — SSE reconnect preserves list state

**Preconditions:** at least two jobs in any status.

**Steps:**
1. Open the App UI tab.
2. Disconnect and reconnect the browser tab (hard refresh).

**Expected:** the live list repopulates from the SSE stream; all previously seen jobs reappear with their current status. No duplicate rows; list is sorted by `createdAt` descending.
