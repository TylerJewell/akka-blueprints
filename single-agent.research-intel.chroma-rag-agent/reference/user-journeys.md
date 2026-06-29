# User journeys — chroma-rag-agent

## J1 — Ask a seeded question and receive a grounded answer

**Preconditions:** Service running on declared port (`http://localhost:9535/`); a valid model-provider API key set, or the mock LLM selected at scaffold time; corpus index built (automatic at startup).

**Steps:**
1. Open `http://localhost:9535/` → App UI tab.
2. From the **Seeded questions** dropdown, pick `Entity lifecycle`.
3. The question textarea fills with: "How does an Akka EventSourcedEntity rebuild its state after a restart?"
4. Click **Ask**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `RETRIEVING` within 1 s. The right-pane detail shows the retrieved chunks accordion with at least 2 chunks whose relevance scores are above 0.80.
- Within 30 s the card reaches `ANSWERED`. The right pane shows: a response text paragraph (1–4 sentences), and a citations table with at least one row. Every `chunkId` in the citations table matches a chunk in the retrieved-chunks accordion.
- Within 1 s of `ANSWERED`, the card reaches `EVALUATED` and shows a groundedness score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a hallucinated citation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-question.json` includes entries with a `chunkId` that does not appear in the top-5 retrieved chunks.

**Steps:**
1. Submit any seeded question three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/queries/sse`).

**Expected:**
- The third submission's first agent iteration produces an answer citing a fabricated `chunkId`.
- The `before-agent-response` guardrail rejects it. The hallucinated citation NEVER lands in `QueryEntity` — there is no `AnswerRecorded` event with the fabricated payload.
- The agent loop retries on iteration 2 and produces a well-formed answer whose every `chunkId` appears in the retrieved context. The card transitions to `ANSWERED` with a valid answer.
- The service log shows one `guardrail.reject` line per rejected iteration naming the offending `chunkId`.

## J3 — Out-of-corpus question scores groundedness 1

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the **Question** textarea, type: "What is the capital of France?"
2. Click **Ask**.

**Expected:**
- The workflow retrieves top-5 chunks; none have a relevance score above 0.40 for this off-topic question.
- The agent returns a refusal: "The indexed corpus does not contain information about European geography." with an empty citations list.
- The guardrail accepts the empty citations list because the response text matches the known refusal pattern.
- The groundedness score is **1** with rationale "No citations provided; answer is a refusal with no retrieved-passage support."
- The card's border highlights red. The user knows to consult an external source.

## J4 — Citation chunkIds match retrieved context (audit check)

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded `Workflow timeouts` question.
2. Wait for `EVALUATED`.
3. Inspect the entity via `GET /api/queries/{id}`.
4. Compare `answer.citations[*].chunkId` against `retrievedChunks[*].chunkId` in the response JSON.

**Expected:**
- Every `chunkId` in `answer.citations` appears in `retrievedChunks`. The sets are not disjoint; no citation references a chunk that was not retrieved.
- `answer.chunksRetrieved` equals the length of `retrievedChunks`.
- `groundedness.score` is 3 or higher (the workflow-timeout topic has good corpus coverage).

## J5 — Multi-chunk answer covers the top passages

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded `View query limitations` question.
2. Wait for `EVALUATED`.

**Expected:**
- The retrieved chunks include at least 3 entries with relevance score above 0.70.
- The answer cites at least 2 of the top-3 chunks (i.e., the agent used the most relevant passages, not just the first one).
- `groundedness.score` is 4 or 5.
- No citation's `passage` field is a paraphrase — the text matches the verbatim passage from the corresponding chunk in `retrievedChunks`.
