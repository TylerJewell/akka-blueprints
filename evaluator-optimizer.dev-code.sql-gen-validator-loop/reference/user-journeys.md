# User journeys — sql-gen-validator-loop

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9335; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9335/`. App UI tab is visible.
2. In the Question field, type "Show me the top 10 customers by total order value". Leave the Schema field at `default_schema`. Click Submit.
3. A new request card appears with status `GENERATING`.

**Expected:**
- Within 1 s, the first attempt's generated SQL appears.
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's validation is `VALID` and the request transitions to `ACCEPTED`, OR
  - Attempt 1's validation is `INVALID` with 1–3 bullets and the request transitions back to `GENERATING`, with attempt 2 appearing shortly after; this continues until either a `VALID` or the retry ceiling is reached.
- On `ACCEPTED`, the terminal block shows the accepted SQL and "best of N attempts."
- The expanded view shows every attempt's SQL, guardrail verdict, validator verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Validator to always return `INVALID` (test mode — submit the literal question `"test-force-invalid"`, which the mock provider's seedFor logic always answers with `INVALID`).

**Steps:**
1. Submit the question `"test-force-invalid"` with the schema `default_schema`.

**Expected:**
- Request progresses `GENERATING` → `VALIDATING` → `GENERATING` → `VALIDATING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `INVALID`, the request transitions to `FAILED_FINAL` (not stuck in `VALIDATING`).
- The terminal block shows the highest-scoring attempt's SQL as the "best of 4 attempts" and the `failureReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its SQL, guardrail OK verdict, INVALID validation result, and score.
- `GET /api/queries/{id}` returns the full QueryRequest with all 4 attempts in `attempts[]` and `status: "FAILED_FINAL"`.

## J3 — Mutation guardrail block

**Preconditions:** As J1.

**Steps:**
1. Submit the question `"delete all orders older than 90 days"` with schema `orders_schema`.

**Expected:**
- The GeneratorAgent produces a query containing `DELETE FROM orders`. The guardrail records `verdict.passed = false`, `reasonCode = "MUTATION_FORBIDDEN"`, with detail naming the forbidden keyword.
- The ValidatorAgent is NOT called for that attempt. The request stays in `GENERATING`.
- The GeneratorAgent is called again with structured feedback (`"Query contains a mutating keyword; rewrite as a SELECT-only statement."`). The revised attempt produces a `SELECT` query identifying old orders without deleting them.
- The validator scores the revised query normally. The loop continues until `VALID` or the retry ceiling.
- The expanded view shows the blocked attempt with the red `MUTATION_FORBIDDEN` pill and no validator verdict, then the revised attempt with `OK` and a validator result.

## J4 — Eval-event timeline

**Preconditions:** At least one request has completed (any terminal state).

**Steps:**
1. Click the request card to expand.

**Expected:**
- The timeline shows one `ValidationEvalRecorded` event per validated attempt, with `verdict`, `score`, and `mutationBlocked` populated.
- The terminal transition (QueryRequestAccepted or QueryRequestFailedFinal) is also surfaced as a final `ValidationEvalRecorded` event carrying the loop-level outcome.
- `GET /api/queries/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
