# User journeys — multi-step-query-engine

## J1 — Submit a seeded question and get a scored answer

**Preconditions:** Service running on declared port (`http://localhost:9201/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded question `What are the primary drivers of transformer model scaling laws?` has a matching `src/main/resources/sample-data/questions/transformer-scaling-laws.json` file.

**Steps:**
1. Open `http://localhost:9201/` → App UI tab.
2. From the **Pick a seeded question** dropdown, pick `What are the primary drivers of transformer model scaling laws?`.
3. Click **Run query**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `DECOMPOSING` within 1 s more.
- Within ~20 s the card reaches `DECOMPOSED`. The right pane shows the Sub-questions panel with 2–4 entries; each entry has a non-empty `text` and a numeric `priority`.
- Within ~20 s more the card reaches `RETRIEVED`. The right pane shows the Passages table with ≥ 2 rows per sub-question; each row has a non-empty source, url, text, and a relevance score between 0.0 and 1.0.
- Within ~20 s more the card reaches `SYNTHESIZED`, then `EVALUATED` within 1 s of that. The right pane shows a `QueryAnswer` with `sections.length == subQuestions.length`, every section has at least one cited passage id, every cited passage id exists in the retrieved evidence set, and the eval score chip shows ≥ 4/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Hallucinated citation flags eval score ≤ 2

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `synthesize-answer.json` includes one entry whose first `AnswerSection.citedPassageIds` references a passageId absent from the paired `EvidenceSet`.

**Steps:**
1. Submit any seeded question until the hallucinated-citation entry is selected by the mock's `seedFor(queryId)` modulo (typically within 6 submissions).
2. Watch the live list until a card's border highlights red.

**Expected:**
- The flagged query lands well-formed (the pipeline does not crash; the answer is recorded).
- The eval score chip shows **1** or **2** and the rationale reads: *"Citation provenance failed: section 'X' cites passageId 'p-xxxxxxxx' which does not appear in the retrieved EvidenceSet."*
- The card's border highlights red. The researcher knows to inspect this answer before relying on it.
- Other queries in the run scored ≥ 4.

## J3 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider (real or mock).

**Steps:**
1. Submit any seeded question.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the queryId.

**Expected:**
- The DECOMPOSE task's log entries show only `generateSubQuestions` and `rankSubQuestions` calls.
- The RETRIEVE task's log entries show only `searchPassages` and `fetchPassage` calls.
- The SYNTHESIZE task's log entries show only `draftSection` and `composeAnswer` calls.
- No cross-phase calls appear.
- The order of tasks in the log is DECOMPOSE → RETRIEVE → SYNTHESIZE. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J4 — Custom question with no matching passage file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/passages/<custom-question-slug>.json` exists for the user's question.

**Steps:**
1. In the App UI, type a custom research question (e.g. `What factors drive spontaneous order formation in ant colonies?`) into the question field.
2. Click **Run query**.

**Expected:**
- `RetrieveTools.searchPassages` returns an empty list for every sub-question.
- The agent's RETRIEVE task returns an `EvidenceSet` with `passages = []`.
- The workflow advances to `synthesizeStep`. The SYNTHESIZE task returns a `QueryAnswer` with `confidence = INSUFFICIENT`, `sections = []`, and a one-sentence summary explaining the gap.
- The eval score chip shows 1 (no checks satisfy with zero passages). The rationale names "no evidence passages retrieved."
- The pipeline completes without error; the empty answer is honestly empty.

## J5 — Confidence calibration check

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded question `How does retrieval-augmented generation compare to fine-tuning for domain adaptation?` (whose mock-paired `EvidenceSet` contains exactly one passage per sub-question).
2. Wait for `EVALUATED`.

**Expected:**
- The recorded `QueryAnswer.confidence` is `MEDIUM` (exactly 1 passage per sub-question satisfies the MEDIUM threshold but not HIGH).
- The eval score is ≥ 3 (sub-question coverage, citation provenance, and section parity hold; confidence calibration holds for MEDIUM with 1 passage per sub-question).
- If the agent's synthesize task incorrectly returned `confidence = HIGH`, the `StoppingEvaluator` would deduct 1 point for confidence miscalibration and the rationale would name "confidence HIGH requires ≥ 2 passages per sub-question, saw 1."

## J6 — Section parity enforced by on-decision eval

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded question `What evidence exists for emergent capabilities in large language models?` (whose mock-paired `DecomposedQuestion` carries 3 sub-questions).
2. Wait for `EVALUATED`.

**Expected:**
- The recorded `QueryAnswer.sections.length` equals the recorded `DecomposedQuestion.subQuestions.length` (both 3).
- Every `section.subQuestionId` matches one `subQuestion.subQuestionId` from the decomposition, one-to-one.
- If the agent's first iteration produced 2 sections instead of 3 (silent collapse), the on-decision evaluator would flag "section parity failed: 2 sections for 3 sub-questions" and deduct 1 point. A score of 5 with rationale "all checks passed" confirms parity held.
