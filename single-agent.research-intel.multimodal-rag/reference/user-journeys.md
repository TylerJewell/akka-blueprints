# User journeys — multiformat-hybrid-rag

## J1 — Submit a seeded question and receive a cited answer

**Preconditions:** Service running on declared port (`http://localhost:9969/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9969/` → App UI tab.
2. From the **Seeded questions** dropdown, pick `Climate science`.
3. The question textarea fills automatically.
4. Click **Submit question**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `RETRIEVING` within 1 s. The right-pane detail shows a compact list of retrieved chunks, including at least one chunk with format `PDF_PAGE` and one with format `TEXT`.
- Within 30 s the card reaches `ANSWERED`. The right pane shows: an `ANSWERED` decision badge, a 2–6-sentence answer paragraph, and a citation table with at least 2 rows. Every citation row's `chunkId` appears in the retrieved chunk list shown above.
- No citation row has an empty `passageExcerpt`.

## J2 — Guardrail blocks a citation-free answer

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-question.json` includes one deliberately malformed entry (a citation `chunkId` not in the retrieved set) selected on the first iteration of every 3rd query.

**Steps:**
1. Submit any seeded question three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/queries/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed answer (a chunkId that is not in the retrieved set for that query).
- The `before-agent-response` guardrail rejects it. The malformed answer NEVER lands in `QueryEntity` — there is no `AnswerRecorded` event with the invalid citation.
- The agent loop retries on iteration 2 and produces a well-formed answer with valid chunkIds. The card transitions to `ANSWERED`.
- The service log shows one `guardrail.reject` line per rejected iteration naming which chunkId failed the cross-reference check.

## J3 — Image-caption chunk is correctly attributed

**Preconditions:** Service running. Any model provider. The seeded corpus includes at least 4 IMAGE_CAPTION chunks across the topic domains.

**Steps:**
1. Submit the `Carbon capture technology` seeded question.
2. Wait for `ANSWERED`.
3. Inspect the right-pane citation table.

**Expected:**
- At least one citation row carries format badge `IMAGE_CAPTION`.
- That row's `passageExcerpt` begins with `"Caption:"` (matching the normalisation applied by `ChunkIndexer`).
- The citation row has the camera-icon visual indicator in the UI.
- The `sourceTitle` on that citation matches the `sourceTitle` of an IMAGE_CAPTION chunk in `corpus-chunks.jsonl`.

## J4 — Out-of-scope question returns NO_RESULT

**Preconditions:** Service running. Any model provider (the mock includes a `NO_RESULT` entry for an out-of-scope question).

**Steps:**
1. In the App UI, pick `Custom` from the seeded questions dropdown.
2. Type a question that is clearly outside the seeded corpus scope, e.g., `"What is the history of the Byzantine Empire?"`.
3. Click **Submit question**.

**Expected:**
- The card transitions to `RETRIEVING`. The retrieved chunk list shows chunks with low relevance scores (all below 0.3).
- The card transitions to `ANSWERING` then `NO_RESULT`. The decision badge shows `NO_RESULT`.
- The right pane shows the `noResultReason` text in muted italic (not empty).
- The citation table is absent or empty.
- The `ResearchAnswer.citations` array in the API response (`GET /api/queries/{id}`) is empty.

## J5 — Multi-format corpus coverage verified

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the `Ocean biodiversity` seeded question.
2. Wait for `ANSWERED`.
3. Inspect the retrieved chunks section in the right pane.

**Expected:**
- The retrieved chunk list includes chunks from at least two distinct `SourceFormat` values (e.g., TEXT and MARKDOWN, or PDF_PAGE and IMAGE_CAPTION).
- Each retrieved chunk row's format badge is correctly coloured (TEXT=muted, MARKDOWN=blue, PDF_PAGE=yellow, IMAGE_CAPTION=purple).
- The citation table's format column matches the format of the corresponding chunk in the retrieval list.
