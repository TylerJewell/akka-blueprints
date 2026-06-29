# User journeys — podcast-summarizer

## J1 — Submit an episode and get a summary

**Preconditions:** Service running on declared port (`http://localhost:9821/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded episode `Deep Cuts: Postgres internals with Bruce Momjian` has a matching `src/main/resources/sample-data/transcripts/deep-cuts-postgres.json` file.

**Steps:**
1. Open `http://localhost:9821/` → App UI tab.
2. From the **Pick a seeded episode** dropdown, pick `Deep Cuts: Postgres internals with Bruce Momjian`.
3. Click **Run pipeline**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `EXTRACTING` within 1 s more.
- Within ~20 s the card reaches `EXTRACTED`. The right pane shows the Extracted-quotes table with ≥ 4 rows; each row has a non-empty speaker, timestampLabel, and text.
- Within ~20 s more the card reaches `SEGMENTED`. The right pane shows ≥ 2 topic clusters; every `cluster.quoteIds` entry matches a `quote.quoteId` from the extracted table.
- Within ~20 s more the card reaches `SUMMARIZED`, then `EVALUATED` within 1 s of that. The right pane shows an `EpisodeSummary` with `sections.length == clusters.length`, every section has at least one `supportingQuoteId`, every `supportingQuoteId` matches a `quote.quoteId`, and the eval score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `extract-quotes.json` includes one entry whose `tool_calls` array starts with a SUMMARIZE-phase tool (`draftSection`) — the deliberately phase-violating entry.

**Steps:**
1. Submit any seeded episode three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/episodes/sse`).

**Expected:**
- On the third submission's `extractStep`, the agent's first iteration calls `draftSection`. `PhaseGuardrail` rejects it; a `GuardrailRejected{phase: "EXTRACT", tool: "draftSection", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `SummarizeTools.draftSection` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal extract sequence (`scanTranscript` + `fetchSpeakerTurn`). The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Fabricated quote id flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `write-summary.json` includes one entry whose first `SummarySection.supportingQuoteIds` contains an id absent from the paired `QuoteSet`.

**Steps:**
1. Submit any seeded episode six times. (The fabricated-quote entry is selected once in every six runs by the mock's `seedFor(episodeId)` modulo.)
2. Watch the live list until a card border highlights red.

**Expected:**
- The flagged episode lands well-formed (the guardrail only checks phase order, not quote provenance).
- The eval score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads: *"Quote provenance failed: section 'X' references quoteId q-XXXX which does not appear in the recorded QuoteSet."*
- The card's border highlights red. The editor knows to inspect this summary before acting.
- The other five episodes in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded episode.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the episodeId.

**Expected:**
- The EXTRACT task's log entries show only `scanTranscript` and `fetchSpeakerTurn` calls.
- The SEGMENT task's log entries show only `clusterQuotes` and `labelCluster` calls.
- The SUMMARIZE task's log entries show only `draftSection` and `writeOverallTakeaway` calls.
- No cross-phase calls appear. (If the mock-LLM violation path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.)
- The order of tasks in the log is EXTRACT → SEGMENT → SUMMARIZE. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Cluster-section parity

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded episode `The Founder's Dilemma with Reid Hoffman` (whose mock-paired `Segmentation` carries 4 clusters).
2. Wait for `EVALUATED`.

**Expected:**
- The recorded `EpisodeSummary.sections.length` equals the recorded `Segmentation.clusters.length` (both 4).
- Every `section.clusterId` matches one `cluster.clusterId` from the segmentation, one-to-one.
- If the agent's first iteration produced 3 sections instead of 4 (silent collapse), the on-decision evaluator would have flagged it with a "section parity" failure and a score of 4. The presence of a score of 5 with rationale "all checks passed" confirms the parity held.

## J6 — Empty transcript with no matching sample data

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/transcripts/<slug>.json` exists for the user's episode title, and `transcriptText` is an empty string.

**Steps:**
1. In the App UI, type a custom episode title (e.g. `Roundtable: the future of artisanal cheese`) into the title field and leave the transcript area blank.
2. Click **Run pipeline**.

**Expected:**
- `ExtractTools.scanTranscript` returns an empty list.
- The agent's EXTRACT task returns a `QuoteSet` with `quotes = []`.
- The workflow advances to `segmentStep`. The SEGMENT task returns a `Segmentation` with `clusters = []`.
- The workflow advances to `summarizeStep`. The SUMMARIZE task returns an `EpisodeSummary` with `overallTakeaway = "(no topic clusters found)"` and `sections = []`.
- The eval score chip shows 1 (cluster count = 0, section count = 0 — parity is satisfied, but no other rule scores). The rationale names "no clusters to score."
- The pipeline completes; nothing crashes; the empty summary is honestly empty.
