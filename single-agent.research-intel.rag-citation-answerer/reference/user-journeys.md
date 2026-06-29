# User journeys — rag-citation-answerer

## J1 — Upload a document and get a cited answer

**Preconditions:** Service running on declared port (`http://localhost:9584/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9584/` → App UI tab.
2. In the Upload panel, click **Load seeded example** and select "Climate Policy Brief".
3. Fill in `Uploaded by` with any string, then click **Upload document**.
4. Watch the document card in the Document list: it transitions `UPLOADED` → `SANITIZED` → `CHUNKED` → `INDEXED` within ~3 s. The card shows the chunk count (e.g., "14 chunks").
5. In the Ask panel, type: "What emission reduction targets does the brief describe?" and check the checkbox next to the uploaded document.
6. Fill in `Asked by` with any string, then click **Ask**.

**Expected:**
- A query card appears in the Query list with status `SUBMITTED` within 1 s, then transitions to `RETRIEVING` within 1 s.
- Within 30 s the card reaches `ANSWERED` with a `HIGH` or `MEDIUM` confidence badge.
- The right-pane detail shows an answer paragraph and a citations table with at least one row. Every citation's `chunkId` matches a chunk from the indexed document. Every citation has a non-empty `excerpt`.
- Clicking a citation row highlights its chunkId in the answer text.

## J2 — Guardrail blocks a hallucinated citation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-question.json` includes deliberately malformed entries — one with a chunkId not present in any retrieved chunk set, one with an empty excerpt.

**Steps:**
1. Upload any seeded document and wait for `INDEXED`.
2. Submit two questions against the same document using the same `askedBy` value.
3. Submit a third question. Watch the third query's lifecycle in the browser dev tools network panel (`/api/queries/sse`).

**Expected:**
- The third query's first agent iteration produces a response with a hallucinated chunkId.
- The `before-agent-response` guardrail rejects it. The malformed answer NEVER lands in `QueryEntity` — there is no `AnswerRecorded` event with the invalid chunkId.
- The agent loop retries on iteration 2 and produces a well-formed answer. The card transitions to `ANSWERED` with valid citations.
- The service log shows one `guardrail.reject` line for the rejected iteration, naming which check failed (`chunkId-not-in-retrieval-set`).

## J3 — PII never reaches the agent

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so chunk content is logged during indexing. Any model provider.

**Steps:**
1. Upload a custom document containing the literal strings `dr.jane.smith@clinic.example.com`, `SSN 078-05-1120`, and `4532-0151-1283-0366` (a test Visa number).
2. Wait for `INDEXED`.
3. Ask any question about the document's non-PII content.
4. Inspect the service log for chunk content logged during indexing (`debug:chunk-indexer.chunk`) and for the agent task attachment logged during `answerStep` (`debug:agent.task.attachment`).
5. Fetch `GET /api/documents/{id}` and inspect `sanitized.redactedText` and `sanitized.piiCategoriesFound`.

**Expected:**
- The indexed chunk content in the log contains only `[REDACTED-EMAIL]`, `[REDACTED-SSN]`, `[REDACTED-PCN]` — not the raw strings.
- The agent task attachment (chunks.json) in the log likewise shows only redacted forms.
- `sanitized.piiCategoriesFound` includes at minimum `email`, `ssn`, and `payment-card-number`.
- The query answer and citations contain no raw PII strings.

## J4 — Empty corpus returns graceful low-confidence answer

**Preconditions:** Service running. No documents uploaded yet (fresh instance).

**Steps:**
1. Open `http://localhost:9584/` → App UI tab.
2. In the Ask panel, type any question. Leave the document checkbox list empty (or skip document selection if the UI prevents submission — use `POST /api/queries` directly with an empty `documentIds` array).
3. Submit the question.

**Expected:**
- The query card appears and transitions to `ANSWERED` (not `FAILED`).
- The answer has `confidence = LOW`, an empty `citations` list, and `answerText` that explains no relevant document chunks were found.
- The UI shows the `LOW` confidence badge and an empty citations table, with no error state shown.

## J5 — Multi-document question with cross-document citations

**Preconditions:** Service running with a real model provider. At least two seeded documents uploaded and reaching `INDEXED`.

**Steps:**
1. Upload the "Climate Policy Brief" and the "Software Procurement Guidance Note" seeded documents.
2. Wait for both to reach `INDEXED`.
3. Ask: "What is the recommended approach to procurement processes according to the uploaded documents?"
4. Check both document checkboxes before submitting.

**Expected:**
- The query card reaches `ANSWERED`.
- The citations table includes at least one row from each of the two selected documents (chunkIds from `doc-<climate-doc-id>-*` and `doc-<procurement-doc-id>-*`).
- No citation references a chunkId from a document that was not in the selected `documentIds` list.
- The `totalChunksSearched` field in `retrieval` equals the combined chunk count of both documents.

## J6 — Agent guardrail exhaustion transitions query to FAILED

**Preconditions:** Service running with the mock LLM. The mock is configured to return malformed entries for all 3 iterations of a specific query seed.

**Steps:**
1. Identify the query seed that triggers all-malformed iterations (documented in mock-responses/answer-question.json comments).
2. Submit a question that hashes to that seed value.

**Expected:**
- All three agent iterations produce malformed answers. The guardrail rejects each.
- After the third rejection, `AnswerWorkflow.answerStep` fails over to the `error` step.
- `QueryEntity` transitions to `FAILED` with a `QueryFailed` event carrying a reason string describing guardrail exhaustion.
- The query card in the UI shows the red `FAILED` status pill and the reason string.
- No partial or malformed `CitedAnswer` is persisted on the entity.
