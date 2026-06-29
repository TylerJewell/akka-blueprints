# User journeys — notion-rag

## J1 — Ask a question and receive a grounded answer

**Preconditions:** Service running on declared port (`http://localhost:9756/`); a valid model-provider API key set and a valid Notion integration token (`NOTION_API_KEY`) with access to a database (`NOTION_DATABASE_ID`), or both mocked at scaffold time.

**Steps:**
1. Open `http://localhost:9756/` → App UI tab.
2. Click the seeded question chip `"Which products support SSO?"`.
3. Leave the session label blank or type a label.
4. Click **Ask**.

**Expected:**
- A new session appears in the left session list within 1 s.
- The question turn appears in the conversation panel with status `SUBMITTED` within 1 s.
- Within 3 s (mock mode) or the Notion API round-trip, the status transitions to `ROWS_RETRIEVED` and the row-retrieval chip shows a count ≥ 1.
- Within 30 s (real model) or 3 s (mock model), the status reaches `ANSWERED`. The answer text is non-empty. The citation table contains at least one row with a `rowId` that matches a row in the retrieved set.
- The answer `confidence` chip is `HIGH` or `MEDIUM` (the seeded schema has three rows with `SupportsSSO = true`).

## J2 — Grounding guardrail blocks an ungrounded answer

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-question.json` includes deliberately malformed entries (a citation with a rowId not in the retrieved set; an empty citations list despite rows being non-empty).

**Steps:**
1. Submit the seeded question "Which products support SSO?" three times in a row.
2. Open the browser dev tools network panel and watch `/api/sessions/{id}/sse`.

**Expected:**
- The third submission's first agent iteration produces an ungrounded answer.
- The `before-agent-response` guardrail rejects it. The ungrounded answer NEVER lands in `SessionEntity` — there is no `AnswerRecorded` event with the ungrounded rowId.
- The agent loop retries on iteration 2 and produces a grounded answer. The conversation panel shows `ANSWERED` with a valid citation table.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming the failed check (`ungrounded-row-id` or `empty-citations`).

## J3 — Zero-row question returns transparent NO_DATA answer

**Preconditions:** Service running. The Notion database returns no rows for the submitted query (mock mode: the mock returns an empty `rows` array for a specific question text; live mode: submit a question that genuinely matches nothing).

**Steps:**
1. Type a question that will match zero rows, e.g., `"Does the database contain quantum computing features?"`.
2. Click **Ask**.

**Expected:**
- The question transitions to `ROWS_RETRIEVED` with a `0 rows retrieved` chip.
- The agent returns `confidence = NO_DATA`, an empty citations list, and an answer text stating no matching rows were found.
- The grounding guardrail passes (zero-row exemption).
- The conversation turn is highlighted amber in the UI. The citation table shows "No matching rows were found."

## J4 — Follow-up question in the same session

**Preconditions:** Service running. A session exists with at least one answered question (J1 completed).

**Steps:**
1. In the right conversation pane of an existing `ACTIVE` session, type a follow-up question: `"What is the pricing for the Enterprise tier?"`.
2. Click **Ask**.

**Expected:**
- The new question turn appears below the existing turn in the conversation panel.
- The session's question count increments by 1.
- The follow-up question goes through the full lifecycle independently (SUBMITTED → ROWS_RETRIEVED → ANSWERING → ANSWERED) without affecting the prior turn's data.
- The `lastActivityAt` timestamp on the session header updates.

## J5 — Notion API token never appears in logs or entity state

**Preconditions:** Service running with a real `NOTION_API_KEY` set and `LOG_LEVEL=DEBUG`.

**Steps:**
1. Submit any seeded question.
2. Wait for `ANSWERED`.
3. Inspect the service log for HTTP outbound calls made by `NotionRetriever`.
4. Fetch `GET /api/sessions/{sessionId}` and inspect the full JSON response.

**Expected:**
- The service log shows the Notion API call with the `Authorization: Bearer ***` header redacted. The raw token value does not appear anywhere in the log output.
- The session JSON response contains no field holding or echoing the Notion API token.
- `retrievedRows` in the response contains only the row data returned by the API, not any credential material.
