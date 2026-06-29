# User journeys — self-correcting-rag

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — End-to-end grounded answer

**Preconditions:** Service running on port 9521; valid model-provider API key set, OR mock mode active; corpus bootstrapped with at least 3 documents.

**Steps:**
1. Open `http://localhost:9521/`. App UI tab is visible.
2. In the Research question field, type "What drives soil carbon loss in drained peatlands?" Click Submit.
3. A new query card appears with status `RETRIEVING`.

**Expected:**
- Within 2 s, status transitions to `GRADING` and the retrieval pass shows the candidate documents with their relevance verdict pills.
- At least one document is `RELEVANT`; the pipeline advances to `GENERATING` without triggering a rewrite.
- Status transitions to `HALLUCINATION_CHECK` as the generated answer appears in the generation block.
- Within 90 s of submission, status transitions to `ANSWERED`; the hallucination verdict pill reads `GROUNDED`.
- The expanded view shows the grounded answer with cited document IDs and the full retrieval pass detail.

## J2 — Query rewrite path

**Preconditions:** As J1; mock mode active with a relevance-grader response that grades all documents `IRRELEVANT` for the trigger query.

**Steps:**
1. Submit the query `"test-force-irrelevant"` (the mock provider's seed logic always returns all-IRRELEVANT for this string on pass 1).

**Expected:**
- Pass 1 grading shows all documents as `IRRELEVANT` (red pills, score < 0.2).
- Status transitions to `REWRITING`; the rewritten query text and rationale appear in the pass 1 block.
- Pass 2 retrieval begins immediately; if the rewritten query produces relevant documents, the pipeline advances to `GENERATING`.
- If pass 2 also yields no relevant documents and `maxRewriteAttempts = 2`, the status transitions to `FAILED_NO_RELEVANT_DOCS` with a structured failure reason.
- `GET /api/queries/{id}` returns the full Query with both retrieval passes in `retrievalPasses[]` and `status: "FAILED_NO_RELEVANT_DOCS"`.

## J3 — Hallucination re-generation

**Preconditions:** As J1; mock mode active with a hallucination-grader response that returns `HALLUCINATED` on the first generation attempt for the trigger query.

**Steps:**
1. Submit the query `"test-force-hallucination"` (the mock provider returns `HALLUCINATED` on generation attempt 1 for this string).

**Expected:**
- Pass 1 retrieval and grading complete normally with at least one `RELEVANT` document.
- Generation attempt 1 produces an answer that the hallucination grader flags as `HALLUCINATED`; the unsupported claim is visible in the UI.
- Status transitions back to `GENERATING` for attempt 2.
- Generation attempt 2 produces a grounded answer; status transitions to `ANSWERED`.
- The expanded view shows two generation blocks — attempt 1 with the `HALLUCINATED` pill and the unsupported claim, attempt 2 with the `GROUNDED` pill and the final answer.

## J4 — Eval-event timeline

**Preconditions:** At least one query has reached `ANSWERED` or a failure state.

**Steps:**
1. Click the query card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per grading step: one for the relevance grading of each retrieval pass, one for each hallucination check.
- Each event shows `stepKind` (`RELEVANCE_GRADE` or `HALLUCINATION_GRADE`), `verdict`, `score`, and `docCount`.
- The terminal transition (`QueryAnswered`, `QueryFailedNoRelevantDocs`, or `QueryFailedHallucination`) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/queries/{id}` includes the eval events (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
