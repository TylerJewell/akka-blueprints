# User Journeys — Financial Model Builder

---

## J1 — Submit ticker and get an approved model (happy path)

**Precondition:** Service running locally on port 9286. Mock filing data present for `AAPL 2025-Q4`.

**Steps:**

1. `POST /api/models` with `{ "ticker": "AAPL", "period": "2025-Q4" }`.
   - Expect: 201 with `{ "modelId": "..." }`.
2. Poll `GET /api/models/{id}` (or watch SSE) until status = `PENDING_REVIEW`.
   - Expect: intermediate statuses EXTRACTING → EXTRACTED → BUILDING → BUILT → VALIDATING → VALIDATED → PENDING_REVIEW in order.
   - Expect: `filingData`, `model`, and `validation` fields all populated.
   - Expect: no `GuardrailRejected` event recorded.
3. Open the App UI at `http://localhost:9286/app`. Navigate to the App UI tab.
   - Expect: the model card appears with status pill PENDING_REVIEW (orange).
   - Expect: Approve and Reject buttons visible in the REVIEW panel.
4. Enter an analyst note and click Approve.
   - Expect: `POST /api/models/{id}/approve` returns 204.
5. Watch status update via SSE.
   - Expect: APPROVED → EVALUATED.
   - Expect: `eval` field populated with `score` (integer 1–5) and `rationale`.
   - Expect: eval score chip visible on the model card.

**Pass criteria:** Entity reaches `EVALUATED`; `EvaluationScored` event recorded; score between 1 and 5.

---

## J2 — Phase-gate guardrail blocks a misordered tool call during EXTRACT

**Precondition:** A test mode or mock agent is configured to attempt a `computeRatio` call (BUILD tool) during the EXTRACT phase.

**Steps:**

1. Submit a model for a ticker in the test mode.
2. The agent receives `EXTRACT_FINANCIALS` task. Before calling `computeRatio`, `ModelPhaseGuardrail` intercepts.
3. Guardrail returns a rejection: tool phase `BUILD` does not match current phase `EXTRACT`.

**Expect:**

- The agent does not execute `computeRatio`.
- The workflow emits `GuardrailRejected` on the entity (audit-only; no state change at that moment).
- The pipeline transitions to `FAILED` via `ModelFailed`.
- `GET /api/models/{id}` returns status `FAILED`.
- The model card in the App UI shows a red dot (guardrail rejection indicator).
- No `FilingData` is written to the entity.

**Pass criteria:** `GuardrailRejected` event present in entity history; status = `FAILED`; `BuildTools.computeRatio` was never called.

---

## J3 — Analyst rejects a model

**Precondition:** A model has reached `PENDING_REVIEW` (J1 steps 1–2 completed).

**Steps:**

1. Open the App UI. The model card shows PENDING_REVIEW (orange).
2. Enter a rejection note: `"Revenue assumptions are not supported by Q4 10-Q data."`.
3. Click Reject.
   - Expect: `POST /api/models/{id}/reject` returns 204.
4. Watch SSE.
   - Expect: status moves to `REJECTED`.
   - Expect: status pill changes to red.
5. `GET /api/models/{id}`.
   - Expect: `analystNote` = `"Revenue assumptions are not supported by Q4 10-Q data."`.
   - Expect: `finishedAt` is populated.
   - Expect: `eval` is absent (null/empty Optional).

**Pass criteria:** `ModelRejected` event in entity history; status = `REJECTED`; `FilingFidelityScorer` was never called; `EvaluationScored` event absent.

---

## J4 — FilingFidelityScorer flags aggressive assumption (score ≤ 2)

**Precondition:** A model is built with mock data configured to include an assumption that deviates more than 20% from the line-item median, and at least one CRITICAL validation flag.

**Steps:**

1. Submit a model for the configured ticker.
2. Wait for `PENDING_REVIEW`.
3. Approve the model via the API or App UI.
4. Wait for `EVALUATED`.

**Expect:**

- `FilingFidelityScorer` runs.
- Check 2 (assumption deviation) fails — `assumedValue` is > 20% from line-item median.
- Check 4 (no CRITICAL flags) fails — at least one CRITICAL flag present in `ValidationReport.flags`.
- Score = 2 (checks 1 and 3 pass; checks 2 and 4 fail).
- `eval.score = 2` in `GET /api/models/{id}`.
- `eval.rationale` names the two failing checks.
- Score chip in the App UI shows `2` (amber or low-score colour).

**Pass criteria:** `EvaluationScored` event with `score = 2`; rationale references assumption deviation and CRITICAL flag; entity status = `EVALUATED`.

---

## J5 (optional) — Ticker with no matching filing data

**Precondition:** Ticker `UNKNOWN` has no mock filing file configured.

**Steps:**

1. `POST /api/models` with `{ "ticker": "UNKNOWN", "period": "2025-Q4" }`.
2. Wait for pipeline to run.

**Expect:**

- `parseFiling("UNKNOWN", "2025-Q4")` returns an empty list.
- Agent returns `FilingData` with `items = []`.
- VALIDATE phase: `ValidationReport.confidence < 60` (no rows to support).
- `ValidationReport.flags` contains at least one WARNING flag noting absent filing data.
- Model reaches `PENDING_REVIEW` (the agent completes; the analyst still decides).
- If analyst approves, `FilingFidelityScorer` score ≤ 2 (checks 1, 2, and 3 fail).

**Pass criteria:** `FilingData.items` is empty; `ValidationReport.confidence < 60`; pipeline reaches `PENDING_REVIEW` without `FAILED`.
