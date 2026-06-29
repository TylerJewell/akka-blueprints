# User journeys — chatbot-sim-eval

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Convergence: simulated user signals resolution

**Preconditions:** Service running on port 9888; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9888/`. App UI tab is visible.
2. Select persona `frustrated-customer`. Enter issue description "I ordered three weeks ago and nothing arrived." Leave max turns at the default 10. Click Submit.
3. A new simulation card appears with status `RUNNING`.

**Expected:**
- Within 1 s, the first user turn appears in the expanded timeline.
- Each chatbot reply shows a guardrail pill (`OK` green).
- Within 120 s of submission, the simulated user sets `signaledResolution = true` on one of its turns. The simulation transitions to `EVALUATING`, then to `PASSED_EVALUATION` or `FAILED_EVALUATION`.
- On `PASSED_EVALUATION`, the terminal block shows the evaluator's summary and a score badge ≥ 7.
- The expanded view shows every turn's customer message, assistant reply, and guardrail status.

## J2 — Evaluation failure

**Preconditions:** As J1, plus mock mode active (which includes a Fail-path evaluator response keyed to `issueDescription = "test-force-fail"`).

**Steps:**
1. Submit persona `first-time-caller`, issue description `"test-force-fail"`, max turns 4.

**Expected:**
- Simulation progresses through up to 4 turns and transitions to `EVALUATING`.
- `EvaluatorAgent` returns `outcome = FAIL` with at least one `DimensionFinding`.
- Simulation status becomes `FAILED_EVALUATION`.
- Terminal block shows the red score badge, the `overallSummary`, and the findings table (Dimension / Score / Observation).
- `GET /api/simulations/{id}` returns the full `Simulation` with all turns in `turns[]`, `verdict.outcome = "FAIL"`, and `status = "FAILED_EVALUATION"`.

## J3 — Guardrail flag on assistant reply

**Preconditions:** As J1, mock mode active (chatbot.json includes one `safeContentFlagged = true` entry triggered by `issueDescription = "test-flag-reply"`).

**Steps:**
1. Submit persona `enterprise-admin`, issue description `"test-flag-reply"`, max turns 6.

**Expected:**
- At least one assistant turn in the timeline shows the `FLAGGED` guardrail pill with a non-empty `flagDetail`.
- The simulation still proceeds to `EVALUATING` — the dialogue is not blocked.
- The evaluator's Policy Compliance dimension score reflects the flag (score ≤ 7 if any turn was flagged).
- `GET /api/simulations/{id}` includes that turn with `safeContentFlagged = true` and the populated `flagDetail`.

## J4 — CI gate

**Preconditions:** At least one simulation in `PASSED_EVALUATION` and one in `FAILED_EVALUATION` are available.

**Steps:**
1. Note the `simulationId` of a `PASSED_EVALUATION` simulation (e.g., `sim-aaa`) and a `FAILED_EVALUATION` simulation (e.g., `sim-bbb`).
2. In the CI gate section of the App UI, enter `sim-aaa,sim-bbb`. Click `Check gate`.
3. Also call `GET /api/simulations/ci-gate?set=sim-aaa,sim-bbb` directly.

**Expected:**
- The banner shows `FAILED` (red) because `sim-bbb` is in `FAILED_EVALUATION`.
- The API response is `{ "passed": false, "failed": ["sim-bbb"], "pending": [] }`.
- Repeat with only `sim-aaa`: banner shows `PASSED` (green); API returns `{ "passed": true, "failed": [], "pending": [] }`.

## J5 — Eval-event timeline

**Preconditions:** At least one simulation has reached a terminal state (any).

**Steps:**
1. Click a completed simulation card to expand.
2. Observe the terminal block.

**Expected:**
- The timeline includes one `EvalRecorded` event for the simulation (emitted by `EvalSampler` within 30 s of the terminal state being reached).
- The `EvalRecorded` event payload shows `outcome`, `overallScore`, and `anyFlagged`.
- `GET /api/simulations/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) containing the same data. The UI renders this without a separate fetch.
