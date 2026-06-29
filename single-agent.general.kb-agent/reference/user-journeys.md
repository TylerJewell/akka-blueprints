# User journeys — kb-agent

Acceptance bar for `/akka:inspect`. Each journey has preconditions, numbered steps, and the expected outcome.

---

## J1 — Grounded answer from seeded corpus

**Precondition:** Service is running with the seeded `kb-documents.jsonl` corpus indexed. No additional documents needed.

**Steps:**
1. Open the App UI tab.
2. Select "policies-1" from the Example questions dropdown. The question "What is the data retention period for customer records?" fills the textarea.
3. Click **Ask**.
4. Observe the card appear in `SUBMITTED` state in the live list.
5. Within ~1 s, the card transitions to `RETRIEVING`; the passage count indicator shows ≥ 1 retrieved passage.
6. Within ~30 s, the card transitions through `ANSWERING` → `ANSWER_RECORDED`. The right pane shows a prose answer and a citations table with at least one row.
7. Within ~1 s of the answer, the card transitions to `EVALUATED`. A groundedness chip with score ≥ 3 appears.

**Expected outcome:** A complete `Query` object with non-null `passages`, `answer`, and `groundedness`. Every `Citation.passageId` in the answer matches a passage in the retrieved list. Groundedness score is 3–5.

---

## J2 — Correct not-found response for out-of-scope question

**Precondition:** Service running with seeded corpus. The question "How do I configure multi-region failover?" has no matching passages in the seeded documents.

**Steps:**
1. Type "How do I configure multi-region failover?" into the Question textarea (or select "custom" from the dropdown and enter the text).
2. Click **Ask**.
3. Observe the card progress to `RETRIEVING`.
4. Observe the right pane's Retrieved Passages section show 0 passages, or passages with relevance score ≤ 0.1.
5. Wait for the card to reach `EVALUATED`.
6. Read the answer text.

**Expected outcome:** The answer contains a NOT_FOUND explanation — no invented facts. `KbAnswer.answerStatus` is `NOT_FOUND`. `citations` is an empty list. The groundedness chip shows score 5 (a correct refusal is fully grounded behaviour). The card border does not turn red.

---

## J3 — Low groundedness score flags an ungrounded answer

**Precondition:** Service running with mock LLM enabled (option-a path). The mock includes a `PARTIAL` entry whose answer sentences share no unigram overlap with any retrieved passage excerpt.

**Steps:**
1. Ask any question that triggers the mock's PARTIAL entry (the mock selection is deterministic via `seedFor(queryId)`; use a query whose hash maps to the PARTIAL entry — check `mock-responses/answer-question.json` for which entry is PARTIAL).
2. Wait for the card to reach `EVALUATED`.
3. Observe the groundedness chip score.
4. Observe the card border colour.

**Expected outcome:** Groundedness score ≤ 2. The card border turns red in the live list and in the right pane detail. The groundedness rationale describes the specific issue (missing overlap / invalid citation). The answer is still visible to the user — this is a non-blocking eval.

---

## J4 — New document indexed; subsequent question retrieves from it

**Precondition:** Service running. The question "What is the password rotation policy?" returns NOT_FOUND against the seeded corpus (no matching document exists yet).

**Steps:**
1. Ask "What is the password rotation policy?" and confirm the result is NOT_FOUND.
2. Call `POST /api/documents` with a new document: title "Password Policy v1.0", body containing the text "Passwords must be rotated every 90 days. Service accounts are exempt if using certificate-based authentication.", collection "policies".
3. Wait 1–2 s for `KbDocumentConsumer` to index the document into `KbIndex`.
4. Ask "What is the password rotation policy?" again.
5. Observe the second query's Retrieved Passages section.

**Expected outcome:** The second query retrieves at least one passage from the newly indexed document. The answer is `GROUNDED` and cites the new document. The `CitedSentences` count is ≥ 1.

---

## J5 — SSE stream delivers state transitions in order

**Precondition:** Service running. An SSE client is connected to `GET /api/queries/sse` before the question is submitted.

**Steps:**
1. Connect an EventSource client (or the UI's built-in SSE listener) to `/api/queries/sse`.
2. Submit a question via `POST /api/queries`.
3. Record the sequence of `status` values received in the SSE stream for the resulting `queryId`.

**Expected outcome:** Status sequence is `SUBMITTED → RETRIEVING → ANSWERING → ANSWER_RECORDED → EVALUATED` in that order, with no skipped or repeated states. Each event carries the full `Query` row at the moment of transition. A late-joining client that fetches `GET /api/queries/{id}` after the sequence completes sees the same final state without needing to replay the stream.

---

## J6 — Failed query records partial state and stops cleanly

**Precondition:** Service running with mock LLM. Trigger a workflow failure by setting `maxIterationsPerTask(3)` exhaustion (mock always returning an invalid shape three times in a row — achievable if the mock JSON has a malformed entry selected for a specific queryId seed).

**Steps:**
1. Submit a question that maps to a consistently-malformed mock response (check the mock file for such an entry; or manually POST to an entity that is already in FAILED state to confirm the command is rejected).
2. Wait for the workflow's `answerStep` to exhaust its retry budget and fail over to the `error` step.
3. Observe the card in the UI.

**Expected outcome:** The card shows status `FAILED` with a red pill. The right pane displays the question text and retrieved passages (those were recorded before the failure); `answer` and `groundedness` are null. The entity log preserves `QuerySubmitted` and `PassagesAttached` events for audit. The workflow does not loop indefinitely.
