# User journeys — task-insight-memory

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Verified persistence on first execution

**Preconditions:** Service running on port 9663; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9663/`. App UI tab is visible.
2. In the Task type field, select or type `data-extraction`. In Description, enter "Extract the main numerical statistics from: Revenue grew 14% year-on-year to $2.3B; headcount rose from 4,200 to 4,850." Leave acceptance criteria at default. Click Submit.
3. A new task card appears with status `EXECUTING`.

**Expected:**
- Within 1 s, the task card shows status `EXECUTING` and "Retrieved insights: 0 (cold start)".
- Within 60 s of submission:
  - The `EvaluatorAgent` returns `VERIFIED`; the task card transitions to `EVALUATED` then `VERIFIED`.
  - The Memory store panel gains a new entry under `data-extraction` with provenance pill `VE` and a confidence score.
  - The expanded task detail shows the executor's `answer`, `confidence` gauge, `keyFindings` list, the eval verdict pill (`VERIFIED` green), `qualityScore`, and "Insight persisted: i-…".
- `GET /api/memory?taskType=data-extraction` returns at least one non-superseded insight.

## J2 — Rejection with no memory write

**Preconditions:** As J1, plus mock mode active (or test task type configured to force `REJECTED` from the evaluator). The mock provider returns `outcome=REJECTED` for `taskType=test-force-reject`.

**Steps:**
1. Submit a task with `taskType=test-force-reject`, description "Force rejection for test purposes."

**Expected:**
- Task progresses `PENDING` → `EXECUTING` → `EVALUATED` → `FAILED`.
- The expanded task detail shows the eval verdict pill `REJECTED` (red), `qualityScore < 0.60`, and the rejection bullets in `notes.bullets`.
- "Insight persisted: Not persisted — evaluation rejected." appears in the detail.
- `GET /api/memory?taskType=test-force-reject` returns an empty list; no insight was written.
- `GET /api/tasks/{id}` returns `status: "FAILED"` with `evalVerdict.outcome = "REJECTED"` and no `finishedAt` insight reference.

## J3 — Correction overwrites a prior insight

**Preconditions:** At least one `VERIFIED` insight exists for `taskType=data-extraction` (complete J1 first).

**Steps:**
1. Note the `insightId` of the `data-extraction` insight from J1 (shown in the Memory panel or from `GET /api/memory?taskType=data-extraction`).
2. `POST /api/memory/correction` with body `{ "taskType": "data-extraction", "correctedText": "Always check for implied units; revenue figures may omit the currency symbol.", "supersedes": "<insightId from step 1>" }`.
3. Submit a new `data-extraction` task (same or different description) and wait for `EXECUTING`.

**Expected:**
- `POST /api/memory/correction` returns `201 { "insightId": "i-new…" }`.
- The Memory panel for `data-extraction` shows the new `CO` (CORRECTION) insight and the old insight moves to the superseded list (visible after toggling "show superseded").
- The new task's expanded detail shows the corrected insight in "Retrieved insights used" — not the superseded one.
- `GET /api/memory/{old-insightId}` returns the insight with `supersededAt` populated.

## J4 — Drift alert surfaces in the memory timeline

**Preconditions:** Service has been running long enough that `DriftWatcher` has fired at least once. Alternatively, submit 6 or more `VERIFIED` tasks of the same type (e.g., `data-extraction`) and one of a different type, then wait up to 5 minutes for `DriftWatcher`'s next tick.

**Steps:**
1. Observe the Memory store panel after the next `DriftWatcher` tick (within 5 minutes of accumulating skewed insights).

**Expected:**
- An amber "Drift alert" row appears in the Memory store panel with the assessment timestamp, `DriftReason.concentrationRatio`, `DriftReason.avgConfidence`, and `DriftReason.triggeredThreshold`.
- `GET /api/memory` includes a `DriftAssessmentRecorded` entry (or equivalent surfaced through the view) with the same metrics.
- The UI does not auto-prune any insights; the alert is informational only.
- A second `DriftWatcher` tick within the same 5-minute window does not produce a duplicate event.
