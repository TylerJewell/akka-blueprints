# User journeys — long-rag-workflow

## J1 — Submit a query and get a research report

**Preconditions:** Service running on declared port (`http://localhost:9544/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded query `EU AI Act liability provisions` has a matching `src/main/resources/sample-data/corpus/eu-ai-act.json` file.

**Steps:**
1. Open `http://localhost:9544/` → App UI tab.
2. From the **Pick a seeded query** dropdown, pick `EU AI Act liability provisions`.
3. Click **Run pipeline**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `RETRIEVING` within 1 s more.
- Within ~30 s the card reaches `RETRIEVED`. The right pane shows the Retrieved-chunks table with ≥ 6 rows; each row has a non-empty doc title, page range, chunk id, and text preview.
- Within ~30 s more the card reaches `SYNTHESIZED`. The right pane shows ≥ 2 themes and ≥ 4 findings; every `finding.supportingChunkId` matches a `chunk.chunkId` from the chunks table.
- Within ~30 s more the card reaches `REPORTED`, then `EVALUATED` within 1 s of that. The right pane shows a `ResearchReport` with `sections.length == themes.length`, every section has ≥ 2 citations, every citation chunk id matches a chunk in the window, and the coverage score chip shows 5/5.
- Total elapsed time: ≤ 90 s on the happy path.

## J2 — Citation guardrail flags and revises an uncited passage

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `retrieve-chunks.json` includes one entry whose model response text contains no `[chunkId]` tags — this is the deliberately uncited entry.

**Steps:**
1. Submit any seeded query three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/queries/sse`).

**Expected:**
- On the third submission, the agent's first response on one of its tasks contains an uncited sentence. `CitationGuardrail` holds the response and returns a citation-gap revision instruction; a `CitationFlagged{sentence, missingCitation, windowSize}` event lands on the entity.
- The uncited response NEVER reaches the workflow as a task result — there is no log line showing the uncited text was written to the entity.
- The agent's next iteration produces a response with `[chunkId]` tags on all sentences. The query eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small orange dot indicating a citation flag fired. The citation-flag log strip on the right pane shows the one flagged sentence with its full structured reason.

## J3 — Out-of-window citation flags coverage score

**Preconditions:** Mock LLM mode. The mock's `write-report.json` includes one entry whose first `ReportSection` cites a chunk id absent from the paired `ChunkWindow`.

**Steps:**
1. Submit any seeded query six times. (The out-of-window entry is selected once in every six runs by the mock's `seedFor(queryId)` modulo.)
2. Watch the live list until a query's card border highlights red.

**Expected:**
- The flagged query lands well-formed (the citation guardrail checks sentence-level citation tags, not whether the chunk id is in the window — that check belongs to the evaluator).
- The coverage score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads: *"Chunk provenance failed: section 'X' cites chunkId 'c-...' which does not appear in the recorded ChunkWindow."*
- The card's border highlights red. The reader knows to inspect this report before acting.
- The other five queries in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded query.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines. Filter to entries tagged with the queryId.

**Expected:**
- The RETRIEVE task's log entries show only `searchChunks` and `fetchChunkText` calls.
- The SYNTHESIZE task's log entries show only `extractFindings` and `groupFindings` calls.
- The REPORT task's log entries show only `draftSection` and `gatherCitations` calls.
- No cross-phase calls appear.
- The order of tasks in the log is RETRIEVE → SYNTHESIZE → REPORT. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.
- Any `CitationFlagged` log lines appear as audit events, not as tool calls — they are entity-write operations, not function-tool invocations.

## J5 — Theme-section parity

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded query `NIST AI RMF alignment strategies` (whose mock-paired `Synthesis` carries 4 themes).
2. Wait for `EVALUATED`.

**Expected:**
- The recorded `ResearchReport.sections.length` equals the recorded `Synthesis.themes.length` (both 4).
- Every `section.themeId` matches one `theme.themeId` from the synthesis, one-to-one.
- If the agent's first iteration produced 3 sections instead of 4, the on-decision evaluator would flag it with a "section parity" failure and a score of 4. A score of 5 with rationale "all checks passed" confirms parity held.

## J6 — Custom query with no matching corpus file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/corpus/<slug>.json` exists for the user's query.

**Steps:**
1. In the App UI, type a custom query (e.g. `quantum error correction for beginners`) into the query field.
2. Click **Run pipeline**.

**Expected:**
- `RetrieveTools.searchChunks` returns an empty list.
- The agent's RETRIEVE task returns a `ChunkWindow` with `chunks = []`.
- The workflow advances to `synthesizeStep`. The SYNTHESIZE task returns a `Synthesis` with `themes = []` and `findings = []`.
- The workflow advances to `reportStep`. The REPORT task returns a `ResearchReport` with `title = "(no synthesis themes)"` and `sections = []`.
- The coverage score chip shows 1. The rationale names "no themes to score."
- The pipeline completes; nothing crashes; the empty report is honestly empty.
