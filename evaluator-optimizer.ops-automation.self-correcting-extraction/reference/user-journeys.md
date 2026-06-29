# User journeys — self-correcting-extraction

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the budget cap

**Preconditions:** Service running on port 9796; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9796/`. App UI tab is visible.
2. In the Raw document text field, paste an invoice: `"Invoice #INV-2024-0042, dated 15 March 2024. Vendor: Acme Corp. Total: $1,250.00 USD."` Leave the document type as `invoice`. Click Submit.
3. A new job card appears with status `EXTRACTING`.

**Expected:**
- Within 1 s, status transitions to `EXTRACTING` (already there) and attempt 1's extracted fields appear.
- The scorer's verdict on attempt 1 is either `PASS` (job transitions to `VERIFIED`) or `CORRECT` with 1–4 bullets (job transitions back to `EXTRACTING`, attempt 2 appears shortly after).
- On `VERIFIED`, the terminal block shows the verified field map and a "verified on attempt N" caption.
- The expanded view shows every attempt's field map, scorer decision, confidence score, and correction notes.
- If mock mode is active, the trajectory is deterministic per `(jobId, attemptNumber)` via `MockModelProvider.seedFor`.

## J2 — Budget exhausted

**Preconditions:** As J1, plus an override that forces the Scorer to always return `CORRECT` (test mode — submit the document type `"test-force-exhaust"`, which the mock provider's seedFor logic always answers with `CORRECT`).

**Steps:**
1. Submit a document with `documentType = "test-force-exhaust"` and any non-blank `rawText`.

**Expected:**
- Job progresses `EXTRACTING` → `SCORING` → `EXTRACTING` → `SCORING` → … for `budgetCap` cycles (default 4).
- After the 4th cycle ends in `CORRECT`, the job transitions to `BUDGET_EXHAUSTED` (not stuck in `SCORING`).
- The terminal block shows the highest-confidence attempt's field map as best-of and `exhaustionReason` reads `"budget cap reached (4)"`.
- All 4 attempts are present in the expanded view, each with its field map, `CORRECT` scorer decision, and correction notes.
- `GET /api/jobs/{id}` returns the full ExtractionJob with all 4 attempts in `attempts[]` and `status: "BUDGET_EXHAUSTED"`.

## J3 — Memory recall across jobs

**Preconditions:** At least one job with `documentType = "invoice"` has completed as `VERIFIED`.

**Steps:**
1. Submit a second invoice document that contains the same vendor but with a slightly different spelling (e.g., `"ACME CORPORATION"` vs. confirmed `"Acme Corp"`).

**Expected:**
- The Scorer's correction notes for the second job include a "Memory conflict:" bullet naming `vendorName`, the extracted value, and the confirmed value from memory.
- `GET /api/memory/invoice` returns the confirmed field-value pairs from the first job, including `vendorName = "Acme Corp"`.
- After the second job completes as `VERIFIED`, `GET /api/memory/invoice` shows updated `confirmedAt` timestamps for any fields that were re-confirmed.

## J4 — Eval-event timeline

**Preconditions:** At least one job has completed (any terminal state).

**Steps:**
1. Click the job card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per scored attempt, with `decision`, `confidence`, and `memoryHit` populated.
- The terminal transition (`JobVerified` or `JobBudgetExhausted`) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/jobs/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.

## J5 — Window-buffer correction context

**Preconditions:** As J1, with mock mode selecting a partial-extraction response for attempt 1 and a CORRECT verdict.

**Steps:**
1. Submit an invoice document. Allow attempt 1 to receive a `CORRECT` verdict.

**Expected:**
- Attempt 2's call to `CORRECT_EXTRACTION` carries the window buffer containing attempt 1's `FieldMap`.
- If a field was wrong in attempt 1 and is still wrong in attempt 2, the ExtractionAgent sets its value to `"AMBIGUOUS: <reason>"` and lowers `confidence` (observable in the field map table on attempt 2).
- The scorer grades the AMBIGUOUS field and adds a bullet noting the persistent error.
