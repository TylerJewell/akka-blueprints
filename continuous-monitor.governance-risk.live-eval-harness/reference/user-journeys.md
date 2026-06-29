# User journeys — live-eval-harness

## J1 — Decision arrives and gets scored

**Preconditions:** Service running on port 9282; valid model-provider API key set; `DecisionPoller` enabled.

**Steps:**
1. Open `http://localhost:9282/` → App UI tab.
2. Wait up to 10 s for the first simulated decision.

**Expected:**
- Decision card appears with status RECEIVED, then transitions to SANITIZED within 1 s. The sanitized input is visible; any stripped fields are shown as `[STRIPPED]`.
- Within 20 s the status reaches EVALUATED (OK or ALARMED). The overall score badge and rubric dimension rows are populated in the detail pane.
- The decision appears in `GET /api/evals` with a non-null `evalResult`.

## J2 — Low-score decision triggers an alarm

**Preconditions:** Service running; at least one simulated decision with `overallScore <= 3` in `agent-decisions.jsonl`.

**Steps:**
1. Wait for a decision whose score is at or below the alarm threshold (default 3).

**Expected:**
- The decision card shows status ALARMED with a red pill.
- `GET /api/evals/{id}` returns `{ "status": "ALARMED", "alarm": { "score": ..., "threshold": 3, "firedAt": "..." } }`.
- An `eval-update` SSE event with `status=ALARMED` is emitted to connected clients.

## J3 — Drift sampler fires and updates drift panel

**Preconditions:** At least 5 scored decisions exist; `DriftSampler` schedule reduced for the test (`DRIFT_SAMPLER_SECONDS=60`).

**Steps:**
1. Wait for `DriftSampler` to tick (60 s in test mode).

**Expected:**
- The drift panel in the right column updates with a `DriftStatus` of OK, WATCH, or ALARM.
- `GET /api/evals/drift` returns a `DriftSnapshot` with `lastAssessedAt` matching the tick time.
- If any dimension mean fell below 3.0, that dimension name appears in `flaggedDimensions` and a `drift-update` SSE event is emitted.

## J4 — Rubric dimensions visible in detail pane

**Preconditions:** A decision with status OK or ALARMED exists.

**Steps:**
1. Click the decision card.

**Expected:**
- Detail pane shows one row per rubric dimension (accuracy, consistency, fairness, groundedness by default).
- Each row has a score badge (1–5) and a one-sentence justification from the agent.
- `overallScore` matches the arithmetic mean of dimension scores, rounded.

## J5 — Sanitization verified (audit check)

**Preconditions:** Service running with debug logging (`LOG_LEVEL=DEBUG`).

**Steps:**
1. Inspect the service log for the `RubricEvalAgent` invocation payload.

**Expected:**
- The log shows the agent input contains `[STRIPPED]` in place of any `userId` or other stripped field.
- `DecisionEntity.incoming.userId` retains the original value in the audit record.
- `DecisionEntity.sanitized.strippedFields` lists the fields that were removed.

## J6 — Drift alarm clears after quality improves

**Preconditions:** `DriftSnapshotEntity` currently in ALARM state. Subsequent decisions arrive with high scores.

**Steps:**
1. Observe the drift panel showing ALARM.
2. Wait for the next `DriftSampler` tick after enough high-scoring decisions accumulate.

**Expected:**
- `DriftStatus` transitions from ALARM to OK.
- A `DriftResolved` event is emitted on `DriftSnapshotEntity`.
- The drift panel badge turns green; `flaggedDimensions` is empty.
