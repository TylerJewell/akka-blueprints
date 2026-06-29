# User journeys — bm25-rag-agent

## J1 — Submit a seeded question and get a grounded answer

**Preconditions:** Service running on declared port (`http://localhost:9617/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9617/` → App UI tab.
2. Click the seeded-question pill: "How does scaled dot-product attention work?".
3. Leave Top-K at the default (5). Click **Ask**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `PASSAGES_ATTACHED` within 1 s. The right-pane detail shows a passage table with ≥ 1 retrieved passage whose title mentions attention or dot-product.
- Within 30 s the card reaches `ANSWER_RECORDED`. The right pane shows: an answer type badge (`DIRECT` or `PARTIAL`), the answer text, and a citation list. Every citation's `passageId` matches a row in the retrieved passages table.
- Within 1 s of `ANSWER_RECORDED`, the card reaches `SCORED` and shows a grounding score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks an ungrounded citation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-question.json` includes a deliberately malformed entry whose citation references a passageId not in the retrieved set.

**Steps:**
1. Submit any seeded question three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/queries/sse`).

**Expected:**
- The third submission's first agent iteration produces an answer citing a non-existent passageId.
- The `before-agent-response` guardrail rejects it. The ungrounded answer NEVER lands in `QueryEntity` — there is no `AnswerRecorded` event with the hallucinated passage id.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a grounded answer. The card transitions to `ANSWER_RECORDED` with an answer whose citations are all within the retrieved set.
- The service log shows one `guardrail.reject` line per rejected iteration with the error code `citation-not-grounded`.

## J3 — Question outside the corpus returns NO_ANSWER

**Preconditions:** Service running. Any model provider. The bundled corpus covers transformer-specific topics and does not cover reinforcement learning from human feedback.

**Steps:**
1. In the App UI's question textarea, type: "Explain the reward model training step in RLHF."
2. Leave Top-K at 5. Click **Ask**.

**Expected:**
- The card reaches `PASSAGES_ATTACHED`. The retrieved passages table shows 5 entries, but their titles and snippets are only tangentially related (BM25 found the least-bad matches).
- The agent returns `answerType = NO_ANSWER` with an `answerText` explaining that the provided passages do not address RLHF reward model training. The citations list is empty.
- The grounding score is 5 — the scorer awards full marks for correctly restraining the answer rather than fabricating citations.
- No `Guardrail.reject` event appears in the log (a well-formed NO_ANSWER with an empty citations list passes the guardrail cleanly).

## J4 — Low grounding score flags a paraphrased citation

**Preconditions:** Mock LLM mode. The mock includes an entry where `quotedFragment` values are paraphrases rather than verbatim passages.

**Steps:**
1. Submit any seeded question whose mock response entry is the paraphrased one (per the mock's seedFor selection, this is the second entry in the response file).
2. Wait for `SCORED`.

**Expected:**
- The answer lands well-formed (the guardrail validates passage ids, not quotation verbatim-ness).
- The grounding score chip shows **1** or **2** because none of the `quotedFragment` values appear verbatim in the corresponding passage bodies.
- The rationale reads something like "Quoted fragments do not appear verbatim in source passages; citations cannot be independently verified."
- The card border highlights red. The user knows to read the source passages before relying on this answer.

## J5 — Top-K variation changes the evidence set

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit "What is the difference between BPE and WordPiece tokenization?" with Top-K = 1.
2. Note the single passage retrieved and the answer's citations.
3. Submit the same question again with Top-K = 8.
4. Note the eight passages retrieved and compare the answer's citations.

**Expected:**
- The Top-K = 1 query retrieves exactly 1 passage. The agent's answer cites at most 1 passage and may return `PARTIAL` if the single passage does not fully answer the question.
- The Top-K = 8 query retrieves 8 passages covering multiple perspectives on both algorithms. The agent's answer is more likely to be `DIRECT` and may cite 2–4 passages.
- In both cases, every cited `passageId` is within the retrieved set; the guardrail never fires.
- The grounding scores differ: the single-passage answer scores based on the verbatim quality of its one citation; the eight-passage answer scores across all citations.
