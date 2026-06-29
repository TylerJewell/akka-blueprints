# User journeys — swebench-evaluator-loop

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9890; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9890/`. App UI tab is visible.
2. In the Repo name field, type `example-org/myrepo`. In Issue number, type `42`. In Description, type `Function parse_date raises ValueError on dates before 1970`. Leave Max iterations at the default 5. Click Submit.
3. A new issue card appears with status `PATCHING`.

**Expected:**
- Within 2 s of submission, the first iteration's patch appears in the expanded view (or the card shows iteration count 1).
- The CI gate verdict pill on iteration 1 reads `OK`.
- Within 120 s of submission, either:
  - Iteration 1's test result is `PASS` and the issue transitions to `SOLVED`, OR
  - Iteration 1's test result is `FAIL` with 1–2 failing test names and the issue transitions back to `PATCHING`; iteration 2 appears shortly after; this continues until either a `PASS` or the iteration ceiling is reached.
- On `SOLVED`, the terminal block shows the accepted patch and "solved in N iterations."
- The expanded view shows every iteration's patch, CI gate verdict, test evaluator verdict, pass/fail counts, and failure summary.

## J2 — Halt at iteration ceiling

**Preconditions:** As J1, plus an override that forces the TestEvaluatorAgent to always return `FAIL` (test mode — submit the literal description `"test-force-exhausted"`, which the mock provider's seedFor logic always answers with `FAIL`).

**Steps:**
1. Submit repo `test-org/force-fail`, issue `999`, description `test-force-exhausted` with the default ceiling.

**Expected:**
- Issue progresses `PATCHING` → `EVALUATING` → `PATCHING` → `EVALUATING` → … for `maxIterations` cycles (default 5).
- After the 5th cycle ends in `FAIL`, the issue transitions to `EXHAUSTED` (not stuck in `EVALUATING`).
- The terminal block shows the highest-pass-count iteration's patch as the "best of 5 iterations" and the `exhaustionReason` reads `"max iterations reached (5)"`.
- All 5 iterations are present in the expanded view, each with its patch, gate OK verdict, FAIL test result, and pass/fail counts.
- `GET /api/issues/{id}` returns the full Issue with all 5 iterations in `iterations[]` and `status: "EXHAUSTED"`.

## J3 — CI gate block

**Preconditions:** As J1.

**Steps:**
1. Submit repo `example-org/myrepo`, issue `100`, description `Remove unused import`. The mock provider's seedFor logic returns a malformed diff (missing `---` / `+++` header) for the first iteration of this issue.

**Expected:**
- Iteration 1's diff triggers `reasonCode = "MALFORMED_DIFF"`. The CI gate records `IterationGateVerdictRecorded` with `passed = false`.
- The TestEvaluatorAgent is NOT called for iteration 1. The issue stays in `PATCHING`.
- The PatchAgent is called again with a structured feedback message. Iteration 2 is a valid diff and passes the gate.
- The test evaluator scores iteration 2 normally. The loop continues until `PASS` or the iteration ceiling.
- The expanded view shows iteration 1 with the malformed diff text and the red `MALFORMED_DIFF` pill, iteration 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one issue has completed (any terminal state).

**Steps:**
1. Click the issue card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per evaluated iteration, with `verdict`, `passCount`, `failCount`, and `gateBlocked` populated.
- The terminal transition (IssueSolved or IssueExhausted) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome: total iterations, peak pass count, final verdict.
- `GET /api/issues/{id}` includes the full iteration list with test results embedded, from which the UI constructs the timeline without a separate fetch.

## J5 — Token budget exhaustion

**Preconditions:** As J1, plus setting `tokenBudget = 1000` (well below the cost of even one agent round-trip in non-mock mode, or via a mock that reports high token usage).

**Steps:**
1. Submit any issue with `tokenBudget = 1000`.

**Expected:**
- After the first iteration's token usage is recorded and the cumulative total exceeds 1000, the workflow transitions to `exhaustStep` before starting iteration 2.
- The issue transitions to `EXHAUSTED` with `exhaustionReason` reading `"token budget exhausted (1000 tokens)"`.
- Only 1 iteration appears in the expanded view (the one that ran before the budget was hit).
