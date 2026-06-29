# User journeys — reflexion-self-critique

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9547; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9547/`. App UI tab is visible.
2. In the Research question field, type "What are the main regulatory approaches to AI transparency in the EU?" Leave the citation floor at the default 2. Click Submit.
3. A new query card appears with status `RESEARCHING`.

**Expected:**
- Within 1 s, the first attempt's answer appears with the citation list.
- The citation guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's reflexion is `PASS` and the query transitions to `RESOLVED`, OR
  - Attempt 1's reflexion is `RETRY` with a reinforcement paragraph and 1–3 focus bullets and the query transitions back to `RESEARCHING`, with attempt 2 appearing shortly after (conditioned on the verbal memory); this continues until either a `PASS` or the retry ceiling is reached.
- On `RESOLVED`, the terminal block shows the resolved answer, the "resolved at attempt N" caption, and the full citation list.
- The expanded view shows every attempt's answer, citation list, guardrail verdict, reflexion verdict, score, and verbal reinforcement note.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Reflexion agent to always return `RETRY` (test mode — submit the literal question `"test-force-exhaust"`, which the mock provider's seedFor logic always answers with `RETRY`).

**Steps:**
1. Submit the question `"test-force-exhaust"` with the default citation floor.

**Expected:**
- Query progresses `RESEARCHING` → `REFLECTING` → `RESEARCHING` → `REFLECTING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `RETRY`, the query transitions to `EXHAUSTED` (not stuck in `REFLECTING`).
- The terminal block shows the highest-scoring attempt's answer as the best-of-4 and the `exhaustionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its answer, `OK` guardrail verdict, `RETRY` reflexion verdict, and score.
- `GET /api/queries/{id}` returns the full Query with all 4 attempts in `attempts[]` and `status: "EXHAUSTED"`.

## J3 — Citation guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the question `"overview of supply chain AI"` with citation floor **3**.
2. Mock mode is configured so attempt 1 returns a `CandidateAnswer` with only 1 citation.

**Expected:**
- Attempt 1's citation count is 1, which is below the floor of 3. The guardrail records `verdict.passed = false`, `reasonCode = "UNDER_CITED"`, with detail "Found 1 citation; minimum is 3."
- The Reflexion agent is NOT called for attempt 1. The query stays in `RESEARCHING`.
- The Actor is called again with structured feedback (the guardrail's `ReflexionNote`). Attempt 2 returns at least 3 citations and passes the guardrail.
- The reflexion agent scores attempt 2 normally. The loop continues until `PASS` or the retry ceiling.
- The expanded view shows attempt 1 with the under-cited answer and the red `UNDER_CITED` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one query has completed (any terminal state).

**Steps:**
1. Click the query card to expand.

**Expected:**
- The timeline shows one `ReflexionRecorded` event per reflected attempt, with `verdict`, `score`, and `citationShortfall` populated.
- The terminal transition (QueryResolved or QueryExhausted) is also surfaced as a final `ReflexionRecorded` event carrying the loop-level outcome.
- `GET /api/queries/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.

## J5 — Verbal memory conditioning

**Preconditions:** As J1, with at least two attempts visible.

**Steps:**
1. Expand a query that went through at least one `RETRY` cycle.

**Expected:**
- The `reinforcementParagraph` from attempt N's reflexion note is visible in the timeline.
- Attempt N+1's answer text visibly addresses at least one of the focus bullets from attempt N's note (e.g., if bullet 1 mentioned a missing policy update, attempt N+1's text or citation list references that update).
- The citation list for attempt N+1 contains at least one `SourceDocument` not present in attempt N's list.
