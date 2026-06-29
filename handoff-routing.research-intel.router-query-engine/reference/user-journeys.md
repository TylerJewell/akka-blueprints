# User journeys — router-query-engine

## J1 — Structured-data question: end-to-end handoff to StructuredDataEngine

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `QuestionSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated question seeded as a structured-data type to drop.

**Expected:**
- The query card appears with status `RECEIVED` and within ~1 s transitions to `ROUTING`.
- Within ~10 s the routing block shows `engineType = STRUCTURED`, `confidence = high`, and a one-sentence reason. The status pill changes to `ROUTED_STRUCTURED`.
- Within ~5 s of routing, the right column populates with the `StructuredDataEngine` answer: a direct answer text, an action chip (`AGGREGATION_RESULT` or `DIRECT_LOOKUP`), and at least one entry in the source references list.
- The status pill transitions to `ANSWERED`. `finishedAt` is set.
- Within ~10 s of the routing decision, the routing score chip shows a number 1–5 with a rationale.

## J2 — Semantic question: end-to-end handoff to SemanticSearchEngine

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a semantic question (the seeded JSONL covers both engine types in rotation).

**Expected:**
- Router emits `engineType = SEMANTIC`. Status `ROUTED_SEMANTIC`.
- `SemanticSearchEngine` returns an `Answer` with a synthesized narrative (likely `CONCEPT_SUMMARY` or `SYNTHESIS`) and a `sourceRefs` list.
- Status `ANSWERED`.
- The `StructuredDataEngine` is never invoked for this query — no answer from it appears in any surface.

## J3 — Ambiguous question: short-circuits to ESCALATED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the ambiguous question to drop.

**Expected:**
- Router emits `engineType = UNCLEAR` with `confidence = low`.
- The workflow terminates immediately with `QueryEscalated`. Status `ESCALATED`.
- Neither engine is invoked; the right column shows a muted "Escalated — no engine invoked" block.
- `escalationReason` is populated with the router's reason.
- A `RoutingScored` event still fires; the score chip appears (typically high, since the classifier was correct to refuse).

## J4 — Routing score appears on every routed question

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new question through the routing step.

**Expected:**
- Within ~10 s of `RoutingDecided`, the per-query routing score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible on the list-row card and broken down in the right column.
- The chip never changes the workflow's flow — a low score does not block the answer from being published.

## J5 — Manually submitted question follows the same path

**Preconditions:** Service running.

**Steps:**
1. In the App UI, locate the manual-question form at the bottom of the left column.
2. Enter a structured-data question (e.g. "How many active users logged in last week?") and click Submit.

**Expected:**
- `POST /api/queries` is called; a `201` response returns a `queryId`.
- Within one SSE tick, a new query card appears at the top of the left column with status `RECEIVED`.
- The query progresses through `ROUTING` → `ROUTED_STRUCTURED` → `ANSWERING` → `ANSWERED` following the same steps as J1.
- The `source` field on the query card shows "manual".
