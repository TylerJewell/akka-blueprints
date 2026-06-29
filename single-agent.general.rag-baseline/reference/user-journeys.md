# User journeys — rag-baseline

## J1 — Submit a seeded question and get a grounded answer

**Preconditions:** Service running on declared port (`http://localhost:9846/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9846/` → App UI tab.
2. From the **Seeded questions** dropdown, pick the first pre-written question (e.g., "What is event sourcing?").
3. Leave `Submitted by` as the default.
4. Click **Ask**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `PASSAGES_RETRIEVED` within 1 s. The right-pane detail shows the retrieved passages list with similarity scores.
- Within 30 s the card reaches `ANSWER_RECORDED`. The right pane shows: the answer prose paragraph and a citation table with at least one row. Every citation row has a non-empty `passageId`, a non-empty `passageExcerpt`, and a non-empty `claimSupported`.
- Within 1 s of `ANSWER_RECORDED`, the card reaches `EVALUATED` and shows a groundedness chip (1–5) plus a one-line rationale.

## J2 — Question outside corpus returns no-evidence answer with score 1

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-question.json` includes one entry with `citations: []` and `usedPassages: 0`.

**Steps:**
1. In the App UI, select **Custom** from the Seeded questions dropdown.
2. Type a question on a topic the seeded corpus does not cover (e.g., "What is the boiling point of tungsten?").
3. Click **Ask**.

**Expected:**
- The card reaches `PASSAGES_RETRIEVED`. The retrieved passages may have low similarity scores (the retriever always returns top-K even if scores are low).
- The answer that lands in `ANSWER_RECORDED` has `answerText` indicating no relevant material was found, and `citations: []` with `usedPassages: 0`.
- The eval scores it **1** with a rationale such as "Citations list is empty; answer is not anchored in retrieved passages."
- The card border highlights red. The groundedness chip shows 1.

## J3 — Well-cited answer scores 4 or 5

**Preconditions:** Mock LLM mode. The mock returns an entry where every cited passageId exists in the retrieved set and the citation count matches `usedPassages`.

**Steps:**
1. Submit any seeded question with mock LLM.
2. Wait for `EVALUATED`.
3. Check the groundedness chip.

**Expected:**
- `GroundednessEvaluator` confirms all cited passage ids are present in `RetrievedPassages.passages`.
- The groundedness chip shows **4** or **5** (depending on whether ≥ 3 passages were cited).
- The chip is coloured green. No red border on the card.
- The citation table in the right pane shows matching passage excerpts for each citation row.

## J4 — SSE stream delivers transitions to a late-joining client

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a question from the App UI.
2. Immediately after clicking **Ask**, open a second browser tab and navigate to `http://localhost:9846/api/queries/sse`.
3. Watch the raw SSE stream in the second tab.

**Expected:**
- The second tab receives `query-update` events for `PASSAGES_RETRIEVED`, `ANSWERING`, `ANSWER_RECORDED`, and `EVALUATED` in order.
- Each event carries the full query row at that moment (not just a delta) — so the second tab could reconstruct the UI state from the last received event alone.
- No events are missed even though the second tab joined after `SUBMITTED`.

## J5 — Fabricated passage id scores 1 or 2

**Preconditions:** Mock LLM mode. One mock entry in `answer-question.json` includes a citation with a `passageId` that does not exist in the seeded corpus (a fabricated id).

**Steps:**
1. Submit queries until the mock selects the fabricated-id entry (every 4th query by seed — see Section 11 mock spec).
2. Wait for `EVALUATED`.

**Expected:**
- `GroundednessEvaluator` detects that at least one `Citation.passageId` is absent from `RetrievedPassages.passages`.
- The groundedness chip shows **1** or **2** with a rationale such as "One or more cited passage ids were not in the retrieved set."
- The card border highlights red.
- The citation table still renders the fabricated id row (the UI does not hide it) — the user can see which id was rejected.
