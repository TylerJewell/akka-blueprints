# User journeys — code-eval-loop

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the budget ceiling

**Preconditions:** Service running on port 9241; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9241/`. App UI tab is visible.
2. In the Problem statement field, type "Write a Java method that reverses a string without using StringBuilder.reverse()." Leave the language as "java". Click Submit.
3. A new solution card appears with status `GENERATING`.

**Expected:**
- Within 1 s, status transitions to `GENERATING` (already there) and the first attempt's code appears.
- The sandbox verdict pill on attempt 1 reads `OK`.
- Within 90 s of submission, either:
  - Attempt 1's verification is `PASS` (score 100) and the solution transitions to `PASSED`, OR
  - Attempt 1's verification is `FAIL` with 1–5 diagnostics and the solution transitions back to `GENERATING`, with attempt 2 appearing shortly after; this continues until either a `PASS` or the budget ceiling is reached.
- On `PASSED`, the terminal block shows the passed code and "passed on attempt N."
- The expanded view shows every attempt's code, sandbox verdict, verifier verdict, score, and diagnostics.

## J2 — Budget exhaustion

**Preconditions:** As J1, plus an override that forces the Verifier to always return `FAIL` (test mode — submit the literal statement `"test-force-fail"`, which the mock provider's seedFor logic always answers with `FAIL` and score 0).

**Steps:**
1. Submit the problem statement `"test-force-fail"` with language "java".

**Expected:**
- Solution progresses `GENERATING` → `VERIFYING` → `GENERATING` → `VERIFYING` → … for `maxAttempts` cycles (default 5).
- After the 5th cycle ends in `FAIL`, the solution transitions to `EXHAUSTED` (not stuck in `VERIFYING`).
- The terminal block shows the highest-scoring attempt's code as the "best of 5 attempts" and the `exhaustionReason` reads `"max attempts reached (5)"`.
- All 5 attempts are present in the expanded view, each with its code, sandbox OK verdict, FAIL verification, and score.
- `GET /api/solutions/{id}` returns the full Solution with all 5 attempts in `attempts[]` and `status: "EXHAUSTED"`.

## J3 — Sandbox short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the problem statement `"Write a Java method that lists all processes using Runtime.exec"` with language "java".

**Expected:**
- Attempt 1's code contains `Runtime.exec` or `java.lang.Runtime`. The sandbox records `verdict.passed = false`, `reasonCode = "FORBIDDEN_IMPORT"`, with detail naming the offending import and line.
- The Verifier is NOT called for attempt 1. The solution stays in `GENERATING`.
- The Generator is called again with structured feedback listing the forbidden pattern. Attempt 2 does not use `Runtime` and passes the sandbox check.
- The verifier evaluates attempt 2 normally. The loop continues until `PASS` or the budget ceiling.
- The expanded view shows attempt 1 with the code containing the violation and the red `FORBIDDEN_IMPORT` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one solution has completed (any terminal state).

**Steps:**
1. Click the solution card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per verified attempt, with `verdict`, `score`, and `sandboxBlocked` populated.
- The terminal transition (SolutionPassed or SolutionExhausted) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/solutions/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.

## J5 — CI gate enforcement

**Preconditions:** Mock mode active with a verifier mock entry that returns `verdict = PASS` but `score = 60` (simulating a partial-pass hallucination).

**Steps:**
1. Submit any problem statement that routes to the partial-pass mock entry.

**Expected:**
- The workflow receives the VerifierAgent response with `verdict = PASS` and `score = 60`.
- `passStep` rejects the verdict because `score < 100`. It emits `AttemptVerified` with a synthetic `FAIL` carrying the diagnostic `"ci-gate: score was 60, 100 required"` and re-enters `generateStep`.
- The solution does NOT transition to `PASSED` after this attempt.
- The expanded view shows the attempt's verifier verdict as `FAIL` with the ci-gate diagnostic, not as `PASS`.
- The solution continues through subsequent attempts normally until a genuine score-100 PASS or budget exhaustion.
