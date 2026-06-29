# User journeys — rag-pdf-chat

## J1 — Upload a PDF and get a cited answer

**Preconditions:** Service running on declared port (`http://localhost:9192/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9192/` → App UI tab.
2. From the **Load seeded PDF** dropdown, pick `distributed-systems-whitepaper`.
3. Click **Load** to upload the seeded document.
4. Wait for the document card to show status `INDEXED` and a passage count chip (e.g., "24 passages").
5. Select the document, type the question "How does Raft elect a leader?" into the chat input, and click **Ask**.

**Expected:**
- The new exchange card appears in the exchange history with status `RETRIEVING` within 1 s.
- The card transitions to `ANSWERING` within ~1 s. The Citations panel shows 1–5 retrieved passages with passageId chips.
- Within 30 s the card reaches `ANSWERED`. The answer text appears with at least one inline `[P-xxx]` citation marker. Every cited passageId corresponds to a passage card visible in the Citations panel. Clicking a citation marker scrolls the matching card into view.
- Every `CitedPassage.passageId` in the returned answer exists in the Retrieved Passages list.

## J2 — Guardrail blocks a fabricated citation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-question-from-pdf.json` includes a deliberately malformed entry citing passageId `"P-999"`, which is not in any seeded document's passage set.

**Steps:**
1. Upload any seeded document and wait for `INDEXED`.
2. Ask three questions in a row against the same document (J1 steps × 3, varying the question text).
3. Watch the third question's lifecycle in the network panel of the browser dev tools (`/api/documents/{id}/chat/sse`).

**Expected:**
- The third question's first agent iteration produces a `CitedAnswer` citing `"P-999"`.
- The `before-agent-response` guardrail rejects it. The fabricated citation NEVER lands in `ChatSessionView` — there is no `AnswerRecorded` event with `passageId = "P-999"`.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a valid answer. The card transitions to `ANSWERED` with a `CitedAnswer` whose every passageId exists in the retrieved set.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming the failed check.

## J3 — Unanswerable question returns explicit not-found response

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Upload any seeded document and wait for `INDEXED`.
2. Ask a question that cannot be answered by the document, e.g., "What is the GDP of France?" for the `distributed-systems-whitepaper`.

**Expected:**
- The exchange card transitions from `ANSWERING` to `UNANSWERABLE`.
- The `CitedAnswer.answerable` field is `false`.
- The `CitedAnswer.answerText` is exactly `"I cannot find this in the document"`.
- The `CitedAnswer.citations` list is empty.
- The Citations panel shows the "No passages were cited" message card.
- The exchange card shows an `UNANSWERABLE` badge (muted, not an error colour).

## J4 — Subsequent questions re-use the existing index

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Upload the `data-governance-policy-brief` seeded document and wait for `INDEXED`. Note the passage count.
2. Ask question 1: "What data classification tiers does the policy define?"
3. Wait for `ANSWERED`.
4. Ask question 2: "Who is responsible for approving Tier 2 data access?"
5. Wait for `ANSWERED`.

**Expected:**
- The document status remains `INDEXED` after both questions — no additional `DocumentUploaded` or `DocumentIndexed` events appear in the service log.
- Both exchanges have `retrievedPassages` lists drawn from the same passage ids (e.g., `"P-001"` through `"P-018"`); no new passage ids appear.
- The passage count chip on the document card does not change between questions.
- Both answers cite passage ids that exist in the original passage list.

## J5 — Passage attachment is never inlined as instruction text

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged.

**Steps:**
1. Upload any seeded document and wait for `INDEXED`.
2. Ask any question.
3. Inspect the service log for the LLM call body (`debug:agent.task.instructions`).

**Expected:**
- The `instructions` field in the logged LLM call body contains only the question text prefixed with "Question: ". It does not contain any passage text.
- The passage text appears only in the `attachments` field of the logged LLM call body, under `"passages.json"`.
- This confirms `TaskDef.attachment(...)` is used, not string interpolation into the instruction text.
