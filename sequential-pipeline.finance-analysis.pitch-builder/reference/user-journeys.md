# User journeys — pitch-builder

## J1 — Submit a target and get a pitchbook

**Preconditions:** Service running on declared port (`http://localhost:9114/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded target `Meridian Packaging Group` has a matching `src/main/resources/sample-data/targets/meridian-packaging-group.json` file.

**Steps:**
1. Open `http://localhost:9114/` → App UI tab.
2. From the **Pick a seeded target** dropdown, pick `Meridian Packaging Group`.
3. Click **Build pitchbook**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `RESEARCHING` within 1 s more.
- Within ~20 s the card reaches `RESEARCHED`. The right pane shows the Research-items table with ≥ 4 rows; at least one row's metadata field shows a redaction-count badge (≥ 1 PII redacted).
- Within ~20 s more the card reaches `COMPS_READY`. The right pane shows ≥ 3 peer companies and a multiples grid with all six range values populated.
- Within ~20 s more the card reaches `DRAFTED`, then `VALIDATED` within 1 s of that. The right pane shows a `Pitchbook` with a cover page, a non-empty executive summary, ≥ 2 sections, and a validation score chip showing 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — PII sanitizer strips counterparty contact from research

**Preconditions:** Service running. Any model provider. The seeded research file for `Meridian Packaging Group` includes a `RawResearchItem` whose `metadata` field contains `"Contact: analyst.name@firm.example.com"`.

**Steps:**
1. Submit `Meridian Packaging Group` (J1 steps).
2. Wait for `VALIDATED`.
3. Inspect the Research-items table on the right pane and the entity event log.

**Expected:**
- The research item's `metadata` field in the UI shows `Contact: [REDACTED]` — the email address is not visible.
- The green lock icon appears on the pitchbook card in the live list.
- A `PiiRedactionLogged` event is in the entity log with `field = "metadata"`, `patternId = "personal-email"`, and non-zero `offsetStart`/`offsetEnd`.
- The final `Pitchbook.sections[].body` text contains no email addresses.
- The `CompsTable` forwarded to the DRAFT task also contains no email addresses (the sanitized pack is what the workflow forwards).

## J3 — Citation guardrail blocks a hallucinated peer

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `draft-pitchbook.json` includes one entry whose first section cites a ticker (`XYZ`) absent from the recorded `CompsTable.peers`.

**Steps:**
1. Submit any seeded target three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/pitchbooks/sse`).

**Expected:**
- On the third submission's `draftStep`, the agent's first iteration produces a section citing ticker `XYZ`. `CitationValidator` rejects it; a `CitationRejected{sectionId: "comps-analysis", rule: "unknown-ticker", detail: "ticker XYZ not found in CompsTable.peers"}` event lands on the entity.
- The misordered citation NEVER reaches the entity's `DraftWritten` event on the first iteration — only the corrected draft does.
- The agent's second iteration produces a corrected draft citing only recorded tickers. The card eventually reaches `VALIDATED` as in J1.
- The small red dot appears on the card in the live list. The citation-rejection log strip on the right pane shows the one rejected response with its full structured reason.

## J4 — Out-of-range multiple flags validation score ≤ 2

**Preconditions:** Mock LLM mode. The mock's `draft-pitchbook.json` includes one entry whose second section cites a figure `"22.0x EV/EBITDA"` while the recorded `CompsTable.multiples.evEbitdaHigh` is `11.2`.

**Steps:**
1. Submit any seeded target six times. (The out-of-range-multiple entry is selected once in every six runs by the mock's `seedFor(pitchbookId)` modulo.)
2. Watch the live list until a pitchbook's card border highlights red.

**Expected:**
- The flagged pitchbook reaches `VALIDATED` — the before-agent-response guardrail on the first pass may or may not have caught the out-of-range figure depending on the mock path; the `validationStep` scoring pass always catches it.
- The validation score chip shows **≤ 2** and the rationale reads: *"Figure provenance failed: section 'comps-analysis' cites '22.0x EV/EBITDA' which is outside the recorded CompsTable range [7.5, 11.2]."*
- The card border highlights red. The analyst knows to verify the figures before sending to the client.

## J5 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded target.
2. Wait for `VALIDATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`) filtered by `pitchbookId`.

**Expected:**
- The RESEARCH task's log entries show only `searchFilings` and `fetchHeadline` calls.
- The COMPARABLES task's log entries show only `selectPeers` and `buildMultiplesTable` calls.
- The DRAFT task's log entries show only `formatSection` and `buildCoverPage` calls.
- No cross-phase calls appear.
- The PII sanitizer log lines (`debug:pii.sanitizer.redact`) appear between the RESEARCH task's final log line and the `workflow.step.complete` line for `researchStep`, confirming it ran at the task boundary.

## J6 — Custom target with no matching research file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/targets/<custom-target-slug>.json` exists for the user's target.

**Steps:**
1. In the App UI, type a custom target name (e.g. `Northfield Artisan Dairy`) into the target field.
2. Click **Build pitchbook**.

**Expected:**
- `ResearchTools.searchFilings` returns an empty list.
- The agent's RESEARCH task returns a `ResearchPack` with `items = []` and `redactionCount = 0`.
- The PII sanitizer runs but has nothing to redact.
- The workflow advances to `comparablesStep`. `ComparablesTools.selectPeers` returns an empty list (no matching sector data). The COMPARABLES task returns a `CompsTable` with `peers = []`.
- The workflow advances to `draftStep`. The DRAFT task returns a `Pitchbook` with `executiveSummary = "(no comparables available)"` and `sections = []`.
- The validation score chip shows 1 (no peers, no figures, no sections to check). The rationale names "no comparables to score."
- The pipeline completes without crashing; the empty pitchbook is honestly empty.
