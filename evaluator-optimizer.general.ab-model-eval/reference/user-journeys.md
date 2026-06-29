# User journeys — ab-model-eval

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Normal trial with a declared winner

**Preconditions:** Service running on port 9677; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9677/`. App UI tab is visible.
2. In the Task prompt field, type "What is the capital of France?" Leave the preferred candidate at the default A. Click Submit.
3. A new trial card appears with status `RUNNING`.

**Expected:**
- Within 60 s of submission, the trial transitions to `JUDGED`.
- The expanded trial view shows both candidate A and candidate B responses in their respective blocks.
- The per-dimension score bars are populated for both candidates.
- A winner badge declares `A wins`, `B wins`, or `TIE` with the judge's one-sentence rationale.
- `GET /api/trials/{id}` returns the full Trial JSON with `status: "JUDGED"`, both `responseA` and `responseB` populated, and `judgement` non-null.

## J2 — Recertification gate fires

**Preconditions:** As J1. The mock provider is configured so that candidate B wins at least 9 of the last 20 trials (default threshold is ≤ 0.60 win-rate for A in a 20-trial window).

**Steps:**
1. Submit at least 20 task prompts using the simulator (let it run) or manually, with preferred candidate A. Wait for all to complete.
2. Observe whether candidate A's win-rate in the App UI header drops below 60%.
3. Click `POST /api/trials/promote` (or the Promote button in the App UI header, if present).

**Expected:**
- When win-rate drops below threshold, the recertification warning banner appears in the App UI with text indicating the win-rate and the threshold.
- Calling `POST /api/trials/promote` returns `409` with `reason: "recertification-required"`.
- Clicking the `Recertify` button calls `POST /api/trials/recertify` and returns `200 { "cleared": N }` where N ≥ 1.
- After recertify, the warning banner disappears and `POST /api/trials/promote` returns `200 { "promoted": true }`.

## J3 — Timeout path

**Preconditions:** As J1, plus test mode where one candidate is forced to time out (submit the literal prompt `"test-force-timeout-a"`, which the mock provider's `seedFor` logic always answers for candidate A with a zero-token response that triggers the `stepTimeout` failover).

**Steps:**
1. Submit the prompt `"test-force-timeout-a"`.

**Expected:**
- Trial appears with status `RUNNING`.
- Within 65 s, the trial transitions to `TIMED_OUT`.
- The expanded view shows no response blocks; instead a red callout displays the `failureReason` naming `candidateAStep` as the failing step.
- `GET /api/trials/{id}` returns `status: "TIMED_OUT"`, `responseA: null`, `responseB: null`, `judgement: null`, and `failureReason` non-null.
- No `EvalRecorded` event is emitted for timed-out trials (there is no verdict to record).

## J4 — Eval-event timeline

**Preconditions:** At least one trial has completed with status `JUDGED`.

**Steps:**
1. Click the completed trial card to expand.
2. Wait at most 30 s for the `AccuracySampler` to fire.

**Expected:**
- The expanded view shows an `EvalRecorded` event with `winner`, `totalA`, `totalB`, `preferredCandidateWon`, and `recertificationRequired` populated.
- `GET /api/trials/{id}` includes the same information (surfaced through the full Trial JSON or an `evalEvents` array, consistent with the SSE update).
- The `EvalRecorded` event from the workflow's `TrialJudged` transition and the `EvalRecorded` event from the `AccuracySampler` carry the same payload; idempotency ensures there is no duplicate on the entity side.
