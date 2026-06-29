# User journeys — agent-benchmark-harness

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Passing benchmark run

**Preconditions:** Service running on port 9501; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9501/`. App UI tab is visible.
2. Click "Run benchmark now". Leave the category filter set to all categories.
3. A new run card appears with status `PENDING`, then transitions to `RUNNING` within 1 s.

**Expected:**
- As each task completes, the task counter increments (`1/10`, `2/10`, …) and a new task block appears in the expanded view.
- Each task block shows the raw response and the scorer's verdict before the next task begins.
- Within 120 s of triggering, the run reaches either `PASSED` or `FAILED` (mock mode: at least 7/10 tasks pass, landing in `PASSED`).
- On `PASSED`, the terminal block shows the aggregate table with `passRate >= 0.80` and duration in milliseconds.
- `GET /api/runs/{id}` returns the full run with all 10 `taskAttempts` populated and `status: "PASSED"`.

## J2 — Failing run (below threshold)

**Preconditions:** As J1, plus mock mode active with the seed configured so that fewer than 8/10 tasks pass.

**Steps:**
1. Click "Run benchmark now".

**Expected:**
- Run progresses through all tasks; pass count < 8.
- Run transitions to `FAILED` with the aggregate showing `passRate < 0.80`.
- Terminal block shows the aggregate table with red accent and `failureReason` reading `"pass rate N% below threshold 80%"`.
- All task attempts are preserved in the expanded view with their individual scores and rationales.
- `GET /api/runs/{id}` returns the full run with `status: "FAILED"` and `failureReason` populated.

## J3 — CI gate reflects failing run

**Preconditions:** At least one completed `FAILED` run exists (from J2).

**Steps:**
1. Open a terminal and run: `curl http://localhost:9501/api/ci-gate`.

**Expected:**
- Response body: `{ "gated": true, "runId": "<id>", "passRate": <rate>, "threshold": 0.80, "reason": "pass rate N% below threshold 80%" }`.
- HTTP status: `200`.
- The App UI banner at the top of the run list shows `BLOCKED` in red with the same pass rate.
- After triggering a passing run (J1), repeating the curl call returns `{ "gated": false, "runId": "<new id>", "passRate": <rate >= 0.80>, "threshold": 0.80 }`.

## J4 — Accuracy snapshot in the per-run timeline

**Preconditions:** At least one completed run exists; service has been running for at least 15 minutes (or `AccuracySampler` has been manually triggered).

**Steps:**
1. Expand any completed run card in the App UI.
2. Scroll to the bottom of the expanded view.

**Expected:**
- An `AccuracySnapshot` section is visible, showing `passRate`, `passCount`, `failCount`, and `recordedAt`.
- `GET /api/runs/{id}` includes at least one `AccuracySnapshotRecorded` event in the run's event history, with the same field values.
- If the sampler has fired more than once for the same run, only one snapshot event exists (idempotency enforced on `runId`).

## J5 — Category-filtered run

**Preconditions:** As J1.

**Steps:**
1. In the category filter, select only "reasoning". Click "Run benchmark now".

**Expected:**
- The run includes only the 4 tasks with `category: "reasoning"` (as registered in `TaskRegistry`).
- The task counter shows `0/4`, `1/4`, etc.
- `GET /api/runs/{id}` returns a run with exactly 4 `taskAttempts`.
- The aggregate is computed over those 4 tasks only.

## J6 — Empty task filter returns 400

**Preconditions:** As J1.

**Steps:**
1. Send `POST /api/runs { "taskFilter": ["nonexistent-category"] }`.

**Expected:**
- Response: `400 { "error": "No tasks match the specified filter" }`.
- No `RunEntity` or workflow is created.
- The run list in the App UI is unchanged.
