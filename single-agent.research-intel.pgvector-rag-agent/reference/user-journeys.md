# User journeys — pgvector-rag-agent

## J1 — Ask a question and get a cited answer

**Preconditions:** Service running on `http://localhost:9197/`; pgvector container running; a valid model-provider API key set, or the mock LLM selected at scaffold time; seeded corpus documents are pre-loaded.

**Steps:**
1. Open `http://localhost:9197/` → App UI tab.
2. Click **Load seeded question** to fill the question textarea with a question from the seed set (e.g., "How does emptyState() work in an EventSourcedEntity?").
3. Click **Ask**.

**Expected:**
- A new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `RETRIEVING` within 1 s; the right-pane detail shows that passage retrieval is in progress.
- The card transitions to `ANSWERING`. The retrieved passages table populates with up to 5 rows, each showing chunk id, source label, and a relevance score badge.
- Within 30 s the card reaches `ANSWERED`. The right pane shows: the answer text with inline `[chunkId]` chips; a citations list; and the retrieved passages highlighted for any cited chunks.
- Within 1 s of `ANSWERED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Citation guardrail blocks an uncited answer

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-question.json` includes entries with an empty `citations` list.

**Steps:**
1. Submit any seeded question three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/questions/sse`).

**Expected:**
- The third submission's first agent iteration produces an uncited answer (empty `citations`).
- The `before-agent-response` guardrail rejects it. The uncited answer NEVER lands in `QuestionEntity` — there is no `AnswerRecorded` event with the empty-citations payload.
- The agent loop retries on iteration 2 and produces a cited answer. The card transitions to `ANSWERED` with citations that satisfy all three guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code `missing-citations`.

## J3 — Insufficient-context question returns an explicit signal

**Preconditions:** Service running. Any model provider. The seeded question set includes at least one question about a topic absent from the corpus.

**Steps:**
1. Click **Load seeded question** repeatedly until a question about an off-corpus topic loads (identified by the question text, e.g., "What is the latency of Akka Cloud deployment in us-east-1?").
2. Click **Ask**.

**Expected:**
- The card reaches `ANSWERED` with an `answerText` of "The retrieved passages do not contain sufficient information to answer this question."
- The `citations` list is empty; the `CitationGuardrail` passed this response through as a recognised sentinel.
- The eval score chip shows **1** or **2** with a rationale of "No supporting passages found; answer is a correctly-formed insufficient-context signal."
- The card's border highlights red. The researcher knows the corpus does not cover this topic.

## J4 — Ingest a new document and ask about its content

**Preconditions:** Service running. Any model provider. The new document's topic is NOT covered by the three seeded corpus documents.

**Steps:**
1. Open the App UI tab. Expand the **Corpus ingest** panel.
2. Enter a `sourceLabel` (e.g., "akka-timer-reference") and paste a 300-word document body describing Akka timers.
3. Click **Ingest document**. Wait for the document status pill in the corpus list to show `INDEXED` and a chunk count.
4. Type a question whose answer appears in the pasted document (e.g., "How do I schedule a one-shot timer in an EventSourcedEntity?").
5. Click **Ask**.

**Expected:**
- The answer cites at least one chunk from the newly indexed document (`sourceLabel: akka-timer-reference`).
- The retrieved passages table shows at least one row from `akka-timer-reference` with a relevance score ≥ 0.6.
- The eval score is ≥ 3.

## J5 — Fabricated chunk id is caught by the guardrail

**Preconditions:** Mock LLM mode. The mock `answer-question.json` includes one entry where a citation's `chunkId` value ("fake-chunk-999") is not present in any retrieved passage.

**Steps:**
1. Configure the mock to produce the fabricated-citation entry on the next run (or position the question submission so the seed selects it).
2. Submit a seeded question.

**Expected:**
- The guardrail rejects the candidate answer on the first iteration with error code `out-of-context-citation`.
- The `AnswerRecorded` event that eventually lands carries only chunk ids that were present in the retrieved passages.
- The service log records a `guardrail.reject` line naming the fabricated chunk id.

## J6 — Retrieved passages visible before answer completes

**Preconditions:** Service running with a real model provider (real LLM latency needed for this journey).

**Steps:**
1. Submit a question via the UI.
2. Watch the right pane as the card moves from `RETRIEVING` to `ANSWERING`.

**Expected:**
- The retrieved passages table populates with rows as soon as the card enters `ANSWERING` — before the LLM call finishes.
- The researcher can read the context passages while waiting for the answer. This confirms the retrieval and answer steps are sequential (not merged) and that the UI streams intermediate state via SSE.
