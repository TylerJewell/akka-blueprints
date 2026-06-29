# User journeys — corrective-rag-workflow

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Retrieval-sufficient path (no fallback)

**Preconditions:** Service running on port 9220; valid model-provider API key set, OR mock mode active. Corpus pre-seeded with chunks covering AI governance topics.

**Steps:**
1. Open `http://localhost:9220/`. App UI tab is visible.
2. In the Research query field, type "What is the NIST AI RMF GOVERN function?". Leave the relevance threshold at the default 0.7. Click Submit.
3. A new query card appears with status `RETRIEVING`.

**Expected:**
- Status transitions to `EVALUATING` once retrieval completes. The expanded view shows retrieved chunk IDs and scores.
- The relevance verdict pill reads `SUFFICIENT` with average score ≥ 0.7. No guardrail or web-search block appears.
- Status transitions to `ANSWERED` with `AnswerSource = RETRIEVAL_ONLY`.
- The terminal block shows the answer text with cited chunk IDs and no cited URLs.
- No `WEB_SEARCHING` status appears at any point in the timeline.

## J2 — Corrective fallback path (web search fires)

**Preconditions:** As J1. Submit a query whose topic is not present in the in-process corpus.

**Steps:**
1. Submit the query "What did the G7 publish about AI code of conduct in 2023?" with the default threshold.

**Expected:**
- Retrieval completes; the evaluator returns `INSUFFICIENT` (average score < 0.7). Status is `EVALUATING` with the `INSUFFICIENT` pill.
- The guardrail step runs and emits `cleared = true` (the query contains no disallowed pattern). Status transitions to `WEB_SEARCHING`.
- The Web Search Agent fires and returns 3–5 ranked results. Status transitions to `ANSWERED`.
- The terminal block shows `AnswerSource = RETRIEVAL_PLUS_WEB`, the answer text with both cited chunk IDs (if any) and cited URLs, and the web-result list in the attempt's expanded section.
- `GET /api/queries/{id}` returns the full Query with the `webSearch` field populated in attempt 1.

## J3 — Guardrail block (web search suppressed)

**Preconditions:** As J1.

**Steps:**
1. Submit the query "What is the SSN validation rule for AI identity systems?" with the default threshold.

**Expected:**
- Retrieval completes; the evaluator returns `INSUFFICIENT` (the corpus does not contain SSN-specific content).
- The guardrail step runs and detects the pattern `\bSSN\b`. It emits `GuardrailVerdictRecorded` with `cleared = false` and `reasonCode = "DISALLOWED_PATTERN"`.
- The Web Search Agent is **not** called. Status transitions directly to `ANSWER_DEGRADED`.
- The terminal block shows the degradation reason ("Guardrail blocked web search: query matches DISALLOWED_PATTERN") and the best-available retrieved excerpt.
- The expanded view shows the guardrail verdict pill in red with the detail text. No web-result section appears.
- `GET /api/queries/{id}` returns `status: "ANSWER_DEGRADED"` and `answer.source = "DEGRADED_RETRIEVAL"`.

## J4 — Eval-event timeline

**Preconditions:** At least one query has completed (any terminal state).

**Steps:**
1. Click the query card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per completed retrieval attempt, each with `judgment`, `averageScore`, and `webSearchFired` populated.
- The terminal transition (`QueryAnswered` or `QueryDegraded`) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome and the `AnswerSource`.
- `GET /api/queries/{id}` includes the eval events (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
- The `EvalSampler` tick does not create duplicate `EvalRecorded` events; each `(queryId, attemptNumber)` appears exactly once.
