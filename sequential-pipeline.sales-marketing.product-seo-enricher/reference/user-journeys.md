# User journeys — product-seo-enricher

## J1 — Submit a product and get an enrichment

**Preconditions:** Service running on declared port (`http://localhost:9105/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded product `UltraFit Running Shoe X9` has a matching `src/main/resources/sample-data/serp/ultrafit-running-shoe-x9.json` file.

**Steps:**
1. Open `http://localhost:9105/` → App UI tab.
2. From the **Pick a seeded product** dropdown, pick `UltraFit Running Shoe X9`.
3. Click **Run enrichment**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `FETCHING` within 1 s more.
- Within ~20 s the card reaches `FETCHED`. The right pane shows the SERP-entries table with ≥ 5 rows; each row has a non-empty title, url, position, and snippet.
- Within ~20 s more the card reaches `ANALYZED`. The right pane shows ≥ 3 keyword candidates and ≥ 1 competitor signal; every `keywordCandidate.sourceUrl` matches a `serpEntry.url` from the fetched table.
- Within ~20 s more the card reaches `ENRICHED`, then `EVALUATED` within 1 s of that. The right pane shows a `ProductEnrichment` with ≥ 1 ranked keyword, a competitor summary, a recommended meta description ≤ 160 characters, and the quality score chip showing 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Browser-tool phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `fetch-serp.json` includes one entry whose `tool_calls` array starts with an ENRICH-phase tool (`rankKeywords`) — the deliberately phase-violating entry.

**Steps:**
1. Submit any seeded product three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/enrichments/sse`).

**Expected:**
- On the third submission's `fetchStep`, the agent's first iteration calls `rankKeywords`. `BrowserGuardrail` rejects it; a `GuardrailRejected{phase: "FETCH", tool: "rankKeywords", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `EnrichTools.rankKeywords` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal fetch sequence (`fetchSerpPage` + `fetchResultSnippet`). The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Ungrounded ranked keyword flags quality score 1

**Preconditions:** Mock LLM mode. The mock's `write-enrichment.json` includes one entry whose first `RankedKeyword` references a keyword absent from the paired `SerpAnalysis.keywords`.

**Steps:**
1. Submit any seeded product six times. (The ungrounded-keyword entry is selected once in every six runs by the mock's `seedFor(enrichmentId)` modulo.)
2. Watch the live list until an enrichment's card border highlights red.

**Expected:**
- The flagged enrichment lands well-formed (the guardrail only checks phase order, not keyword grounding).
- The quality score chip shows **1** (or 2 depending on which other rules pass) and the rationale reads something like: *"Keyword coverage failed: ranked keyword 'outdoor footwear' does not appear in the recorded SerpAnalysis.keywords."*
- The card's border highlights red. The content editor knows to review this enrichment before publishing.
- The other five enrichments in the run scored ≥ 4.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider.

**Steps:**
1. Submit any seeded product.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the enrichmentId.

**Expected:**
- The FETCH task's log entries show only `fetchSerpPage` and `fetchResultSnippet` calls.
- The ANALYZE task's log entries show only `extractKeywords` and `identifyCompetitors` calls.
- The ENRICH task's log entries show only `rankKeywords` and `writeMetaDescription` calls.
- No cross-phase calls appear. (If the mock-LLM violation path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.)
- The order of tasks in the log is FETCH → ANALYZE → ENRICH. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Custom product with no matching SERP fixture file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/serp/<product-slug>.json` exists for the user's product.

**Steps:**
1. In the App UI, type a custom product (e.g. `Artisanal Wooden Desk Lamp`) into the product-name field.
2. Click **Run enrichment**.

**Expected:**
- `FetchTools.fetchSerpPage` returns an empty list.
- The agent's FETCH task returns a `SerpResult` with `entries = []`.
- The workflow advances to `analyzeStep`. The ANALYZE task returns a `SerpAnalysis` with `keywords = []` and `competitors = []`.
- The workflow advances to `enrichStep`. The ENRICH task returns a `ProductEnrichment` with `rankedKeywords = []`, `competitorSummary = "(no competitors identified)"`, and a short `recommendedMetaDescription` containing only the product name.
- The quality score chip shows 1 (keyword-list non-emptiness fails; the other checks are vacuously satisfied or cannot apply). The rationale names "no ranked keywords produced."
- The pipeline completes; nothing crashes; the empty enrichment is honestly empty.

## J6 — Meta description over length limit flags quality point

**Preconditions:** Mock LLM mode. The mock's `write-enrichment.json` includes one entry whose `recommendedMetaDescription` is longer than 160 characters.

**Steps:**
1. Submit any seeded product. Observe the quality score chip once the card reaches `EVALUATED`.

**Expected:**
- The enrichment is well-formed and keyword-grounded.
- The quality score chip shows 4 (meta-description length check fails; the other three checks pass). The rationale reads: *"Meta-description length failed: description is N characters, exceeds the 160-character limit."*
- The content editor can shorten the description before publishing; the system does not reject the enrichment.
