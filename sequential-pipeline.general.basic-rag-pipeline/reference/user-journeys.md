# User journeys — basic-rag-pipeline

## J1 — Submit a question and get a grounded answer

**Preconditions:** Service running on declared port (`http://localhost:9886/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded question `What are the key principles of transformer architectures?` has a matching document set in `src/main/resources/sample-data/corpus/transformer-architecture.json`.

**Steps:**
1. Open `http://localhost:9886/` → App UI tab.
2. From the **Pick a seeded question** dropdown, pick `What are the key principles of transformer architectures?`.
3. Click **Ask**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `INDEXING` within 1 s more.
- Within ~20 s the card reaches `INDEXED`. The right pane shows the Indexed-corpus panel with a document count ≥ 3 and a non-zero chunk count.
- Within ~20 s more the card reaches `ANSWERING`, then `ANSWERED` within ~20 s of that.
- The right pane shows a `RagAnswer` with non-empty `answerText`, ≥ 1 citation, and every citation's `sourceUrl` appearing in the corpus panel.
- The guardrail chip reads **PASSED**. Total elapsed time: ≤ 45 s on the happy path.

## J2 — Citation-grounding guardrail blocks an ungrounded answer

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-query.json` includes one entry whose first `Citation` references a URL absent from the indexed corpus — the deliberately ungrounded entry.

**Steps:**
1. Submit any seeded question four times in a row (J1 steps × 4).
2. Watch the fourth submission's lifecycle in the network panel of the browser dev tools (`/api/sessions/sse`).

**Expected:**
- On the fourth submission's `queryStep`, the agent's first iteration produces an answer with an ungrounded citation. `AnswerGuardrail` rejects it; a `GuardrailApplied{verdict: "BLOCKED", reason: "citation-not-grounded: https://... not in indexed corpus"}` event lands on the entity.
- The ungrounded answer NEVER reaches the caller — there is no `AnswerDrafted` event for the blocked draft.
- The agent's second iteration produces a grounded answer. The session eventually reaches `ANSWERED` with a `PASSED` guardrail chip.
- The right pane's guardrail section shows `PASSED` on the final state; the SSE stream showed a transient `BLOCKED` event during the retry.

## J3 — No relevant chunks returns an honest empty answer

**Preconditions:** Service running with any model provider. The question does not match any document in the in-process corpus.

**Steps:**
1. In the App UI, type `What is the melting point of tungsten carbide?` into the question field.
2. Click **Ask**.

**Expected:**
- The session progresses through `INDEXING → INDEXED → ANSWERING`.
- `QueryTools.retrieveChunks` returns an empty list (no keyword overlap with any indexed chunk).
- The agent returns `RagAnswer { answerText: "No relevant content found in the indexed corpus.", citations: [] }`.
- The guardrail accepts the empty-citation answer (there are no citations to invalidate).
- The session reaches `ANSWERED` with a `PASSED` guardrail chip and an answer text that honestly names the gap.
- Nothing crashes; the corpus remains intact for subsequent queries.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit the seeded question `How does retrieval-augmented generation differ from fine-tuning?`.
2. Wait for `ANSWERED`.
3. Inspect the service log for tool-call lines. Filter to entries tagged with the sessionId.

**Expected:**
- The INGEST task's log entries show only `loadDocuments` and `indexChunks` calls. No `retrieveChunks` or `buildContext` lines appear during this phase.
- The QUERY task's log entries show only `retrieveChunks` and `buildContext` calls. No `loadDocuments` or `indexChunks` lines appear during this phase.
- The order of tasks in the log is INGEST → QUERY. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.
- The log confirms the dependency contract: the QUERY task's instructions reference the chunk count from the INGEST task's recorded `IndexedCorpus`.

## J5 — Answer is blocked after all iterations are exhausted

**Preconditions:** Service running with the mock LLM in a mode where all `answer-query.json` entries for a given sessionId carry ungrounded citations. (Force this by editing the mock to always select the ungrounded entry.)

**Steps:**
1. Submit any seeded question.
2. Watch the session lifecycle in the SSE stream.

**Expected:**
- Every iteration of the `queryStep` produces an ungrounded answer; `AnswerGuardrail` rejects each one.
- After 4 rejected iterations, the workflow step fails over to the `error` step.
- The entity records `AnswerBlocked{reason: "max iterations exhausted"}`, then `SessionFailed`.
- The session status transitions to `BLOCKED` (then `FAILED` on the error step).
- The UI card shows a red `BLOCKED` status pill. The right pane's guardrail section shows `BLOCKED` with the last rejection reason.
- No hallucinated answer ever becomes visible to the user.
