# User journeys — code-search-rag-agent

## J1 — Ask a question and receive a grounded answer

**Preconditions:** Service running on declared port (`http://localhost:9276/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9276/` → App UI tab.
2. From the **Corpus tag** dropdown, pick `akka-http`.
3. In the **Question** textarea, type: `Where is the HTTP server port bound?`
4. Click **Search**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `CHUNKS_SANITIZED` within 1 s. The right-pane detail shows the chunk metadata list; if any secrets were present in the retrieved chunks, a category badge appears (e.g., `api-key`).
- Within 30 s the card reaches `ANSWERED`. The right pane shows: an answer paragraph citing at least one file path, and a references table with one row per cited file. Every reference has a non-empty `filePath`, valid `startLine`/`endLine`, and a non-empty `relevanceBlurb`.
- Within 1 s of `ANSWERED`, the card reaches `GROUNDED` and shows a grounding score chip (1–5) plus a one-line rationale.

## J2 — Secret in a retrieved chunk never reaches the LLM

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body (chunk attachment content) is logged. Any model provider — the sanitizer runs either way.

**Steps:**
1. Submit a query whose `retrievedChunks` includes a chunk whose `content` contains the literal string `apiKey = "sk-live-abc123def456"`.
2. Wait for `ANSWERED`.
3. Inspect the service log for the LLM call attachment payload (`debug:agent.task.attachment`).
4. Fetch `GET /api/queries/{id}` and read `request.retrievedChunks[0].content`.

**Expected:**
- The logged LLM attachment payload contains `[REDACTED-API-KEY]` in place of `sk-live-abc123def456`. The raw key string does not appear anywhere in the model call.
- `request.retrievedChunks[0].content` in the entity JSON still contains the raw string — the audit log preserves it.
- `sanitized.secretCategoriesFound` lists `api-key` (at minimum).

## J3 — Ungrounded citation flags a low grounding score

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `answer-code-question.json` includes entries whose `references[*].filePath` values are not present in any of the attached chunks (e.g., `src/main/java/com/example/NotAFile.java`).

**Steps:**
1. Submit four queries in sequence with any corpus tag and question (J1 steps × 4).
2. Watch the fourth submission's card in the UI.

**Expected:**
- The fourth query's `GroundingScorer` finds one or more `references[*].filePath` values that do not match any `filePath` in `sanitized.redactedChunks`.
- The grounding score chip shows **1** or **2** with a rationale naming the ungrounded paths.
- The card's border highlights red. The developer knows to verify the file paths manually before using the answer.
- The `AnswerRecorded` event is present in the entity (the answer itself is valid); only the grounding annotation indicates the problem.

## J4 — Empty result for a question with no matching chunks

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a query with `corpusTag = "akka-persistence"` and `retrievedChunks = []` (an empty list — simulating a vector-search miss).
2. Wait for `GROUNDED`.

**Expected:**
- The `ChunkSanitizer` receives the empty list, produces a `SanitizedChunks` with empty `redactedChunks` and empty `secretCategoriesFound`, and transitions the entity to `CHUNKS_SANITIZED`.
- `QueryWorkflow.answerStep` calls `CodeSearchAgent` with the question instruction and zero attachments.
- The agent returns a `SearchAnswer` with an `answerText` explaining no relevant chunks were found and an empty `references` list.
- `GroundingScorer` scores the answer **5** (zero citations, nothing to verify as false) with a rationale of "No references cited; empty result is consistent with empty chunk set."
- The UI card shows status `GROUNDED`, grounding score chip 5, and the answer paragraph with its "not found" explanation.

## J5 — Multi-chunk answer with references from multiple files

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a query with `corpusTag = "all"` and question `"How do Akka streams materialize values?"` with at least 5 chunks attached from the seeded `akka-streams` corpus tag.
2. Wait for `GROUNDED`.

**Expected:**
- The answer references at least 2 distinct `filePath` values, each from a different chunk in the attached set.
- Every `references[i].filePath` appears in `sanitized.redactedChunks[*].filePath` — the grounding score is ≥ 3.
- The reference table in the right pane shows one row per cited file with correct `startLine`/`endLine` values drawn from the chunk metadata.
