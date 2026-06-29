# User journeys — competitor-research-pipeline

## J1 — Submit a competitor name and get a profile

**Preconditions:** Service running on declared port (`http://localhost:9901/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. In live mode, `EXA_API_KEY`, `NOTION_API_TOKEN`, and `NOTION_DATABASE_ID` are set. The seeded competitor `Acme Analytics` has a matching `src/main/resources/sample-data/competitors/acme-analytics.json` file.

**Steps:**
1. Open `http://localhost:9901/` → App UI tab.
2. From the **Pick a seeded competitor** dropdown, pick `Acme Analytics`.
3. Click **Run pipeline**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `SEARCHING` within 1 s more.
- Within ~30 s the card reaches `SEARCHED`. The right pane shows the Search-results table with ≥ 4 rows; each row has a non-empty title, url, and excerpt.
- Within ~30 s more the card reaches `SUMMARIZED`. The right pane shows all five profile field rows; each has a non-empty value and a supporting URL that matches a row in the search-results table.
- Within ~30 s more the card reaches `PUBLISHED`, then `EVALUATED` within 1 s. The right pane shows the full `CompetitorProfile` with a valid Notion page URL link and an eval score chip showing 5/5.
- Total elapsed time: ≤ 90 s on the happy path.

## J2 — Phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with the mock LLM selected. The mock's `search-competitor.json` includes one entry whose `tool_calls` array starts with `writeToNotion` — a PUBLISH-phase tool called during the SEARCH phase.

**Steps:**
1. Submit any seeded competitor three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/competitors/sse`).

**Expected:**
- On the third submission's `searchStep`, the agent's first iteration calls `writeToNotion`. `NotionWriteGuardrail` rejects it with a `phase-violation` error; a `GuardrailRejected{phase: "SEARCH", tool: "writeToNotion", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `PublishTools.writeToNotion` — there is no Notion API call and no log line from the tool body.
- The agent's second iteration falls through to a normal search sequence (`searchCompetitor` + `fetchPage`). The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Schema-violation guardrail blocks an incomplete Notion payload

**Preconditions:** Mock LLM mode. The mock's `publish-profile.json` includes one entry whose `buildNotionPage` output omits the `pricing_model` required field in the Notion payload.

**Steps:**
1. Submit any seeded competitor five times. (The schema-violating entry is selected on a deterministic rotation by the mock's `seedFor(competitorId)` modulo.)
2. Watch the live list for a card that shows a red dot without the card border going fully red (the pipeline eventually completes after the retry).

**Expected:**
- On the flagged competitor's `publishStep`, the agent's first `writeToNotion` call is rejected by `NotionWriteGuardrail` with a `schema-violation: required field 'Pricing Model' is missing` error. A `GuardrailRejected` event lands on the entity.
- The incomplete payload NEVER reaches the Notion API.
- The agent's second iteration calls `buildNotionPage` again (or re-assembles the payload), includes `Pricing Model`, and passes validation. `writeToNotion` proceeds; the card reaches `EVALUATED`.
- The rejection-log strip shows the one schema-violation rejection. The eval score chip shows ≥ 4 (the profile fields are all present after the retry).

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded competitor.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines. Filter to entries tagged with the competitorId.

**Expected:**
- The SEARCH task's log entries show only `searchCompetitor` and `fetchPage` calls.
- The SUMMARIZE task's log entries show only `extractFields` and `classifyDomain` calls.
- The PUBLISH task's log entries show only `buildNotionPage` and `writeToNotion` calls.
- No cross-phase calls appear. (If a mock-LLM violation path fires, a single `guardrail.reject` line precedes the agent's retry; the rejected call is logged but never executed.)
- The order of tasks in the log is SEARCH → SUMMARIZE → PUBLISH. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — All five required profile fields present

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded competitor `Beacon DataOps`.
2. Wait for `EVALUATED`.

**Expected:**
- The `ProfileSummary` recorded on the entity contains exactly five `ProfileField` entries, one for each of the declared dimensions: `pricing_model`, `primary_use_case`, `notable_integrations`, `known_differentiators`, `data_residency_stance`.
- No dimension has an empty `value` (entries with no web evidence use `"(not found)"`).
- The eval score chip shows 5/5 if all four scorer checks pass; or a lower score with a rationale naming the specific gap (e.g., "source provenance failed: ProfileField.supportingUrl 'https://...' not found in SearchResultSet").

## J6 — Unknown competitor with no search results

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/competitors/<slug>.json` exists for the input name.

**Steps:**
1. In the App UI, type a custom competitor name not in the seeded list (e.g., `Fictional Startup XYZ`).
2. Click **Run pipeline**.

**Expected:**
- `SearchTools.searchCompetitor` returns an empty list.
- The agent's SEARCH task returns a `SearchResultSet` with `results = []`.
- The workflow advances to `summarizeStep`. The SUMMARIZE task returns a `ProfileSummary` with all five field values set to `"(no results found)"` and `domainClassification = "other"`.
- The workflow advances to `publishStep`. The PUBLISH task builds a Notion page with the `"(no results found)"` values and publishes it. The mock returns a synthetic `NotionPageRef`.
- The eval score chip shows 1 (field coverage has all five fields but source provenance fails because all `supportingUrl` values are placeholders not in the `SearchResultSet`). The rationale names the gap.
- The pipeline completes; nothing crashes; the empty profile is honestly empty.
