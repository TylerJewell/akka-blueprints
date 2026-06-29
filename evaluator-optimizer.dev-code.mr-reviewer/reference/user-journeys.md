# User journeys — mr-reviewer

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9629; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9629/`. App UI tab is visible.
2. In the App UI form, enter project path `acme/backend`, MR IID `100`, target branch `main`, and a multi-file unified diff containing at least two changed files. Click Submit.
3. A new MR card appears with status `RECEIVED`.

**Expected:**
- The sanitizer step fires; the MR transitions to `REVIEWING` within 1 s (diff contains no secrets).
- `ReviewerAgent` produces pass #1. The MR transitions to `GATE_CHECKING`.
- If gate score ≥ 70: the MR transitions to `CI_PASS` and the terminal block shows the accepted review summary.
- If gate score < 70: the MR returns to `REVIEWING` with gate feedback visible; pass #2 appears shortly after. The loop continues until `CI_PASS` or the ceiling is reached.
- The expanded view shows every pass's sanitizer verdict, guardrail verdict, findings, quality score, gate decision, and gate feedback.
- `GET /api/mrs/{id}/ci-signal` returns `{ "signal": "CI_PASS" }`.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus mock mode configured with `seedFor` logic that always returns `GateDecision = REFINE` for the test MR IID `999`.

**Steps:**
1. Submit a webhook with MR IID `999` and any non-secret diff.

**Expected:**
- MR cycles through `REVIEWING` → `GATE_CHECKING` → `REVIEWING` for `maxPasses` passes (default 3).
- After pass 3 ends in `REFINE`, the MR transitions to `CI_FAIL` (not stuck in `GATE_CHECKING`).
- The terminal block shows the highest-scoring pass's summary as the "best of 3 passes" and the CI signal reads `CI_FAIL`.
- All 3 passes are present in the expanded view, each with its sanitizer OK verdict, review findings, and REFINE gate verdict.
- `GET /api/mrs/999/ci-signal` returns `{ "signal": "CI_FAIL" }`.
- `GET /api/mrs/{id}` returns the full MergeRequest with all 3 passes in `passes[]` and `status: "CI_FAIL"`.

## J3 — Sanitizer block

**Preconditions:** As J1.

**Steps:**
1. Submit a webhook whose `diffText` contains the string `BEGIN PRIVATE KEY` embedded in a diff hunk.

**Expected:**
- The sanitizer step fires; `SanitizerVerdictRecorded` is emitted with `passed = false` and `reasonCode = "SECRET_DETECTED"`.
- The MR transitions to `SANITIZER_BLOCKED` immediately.
- No `ReviewerAgent` or `GateAgent` call is made. The App UI shows the orange `SANITIZER_BLOCKED` status pill and the `SECRET_DETECTED` detail banner.
- `GET /api/mrs/{id}` returns `status: "SANITIZER_BLOCKED"` and an empty `passes[]`.
- `GET /api/mrs/{id}/ci-signal` returns `{ "signal": "PENDING" }` (no CI signal is published for blocked MRs — the diff was not evaluated).

## J4 — Output guardrail block

**Preconditions:** Mock mode active; `reviewer.json` includes one entry whose `commentary` field contains a high-entropy hex string of 40 characters.

**Steps:**
1. Submit a clean diff and configure `seedFor` to return the guardrail-triggering mock response for pass #1.

**Expected:**
- `ReviewerAgent` returns pass #1. The guardrail step detects the high-entropy token in `commentary`.
- `ReviewGuardrailVerdictRecorded` is emitted with `reasonCode = "COMMENTARY_REPRODUCES_SECRET"`.
- The MR stays in `REVIEWING` (not `GATE_CHECKING`). `GateAgent` is NOT called for pass #1.
- `ReviewerAgent` is called again with a structured `GateFeedback` note. Pass #2 has clean commentary.
- The expanded view shows pass #1 with the red `COMMENTARY_REPRODUCES_SECRET` guardrail pill; pass #2 with `OK` and normal gate processing.

## J5 — Eval-event timeline

**Preconditions:** At least one MR has completed (any terminal state).

**Steps:**
1. Click the MR card to expand.

**Expected:**
- The timeline shows one `ReviewEvalRecorded` event per completed gate verdict, with `decision`, `gateScore`, and `secretBlocked` populated.
- The terminal transition (`MrCiPassed` or `MrCiFailed`) is also surfaced as a final `ReviewEvalRecorded` event carrying the loop-level outcome.
- `GET /api/mrs/{id}` includes an `evalEvents[]` array (or equivalent field surfaced through the SSE update) with the same content.
- The UI does not require a separate fetch to render the timeline; the full MR JSON delivered via SSE includes the eval events.
