# User journeys — financial-advisor-pipeline

## J1 — Submit a financial query and receive a full report

**Preconditions:** Service running on `http://localhost:9576/`; a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded query `retirement portfolio rebalancing for age 55` has matching sample-data files under `src/main/resources/sample-data/market/`.

**Steps:**
1. Open `http://localhost:9576/` → App UI tab.
2. From the **Pick a seeded query** dropdown, pick `retirement portfolio rebalancing for age 55`.
3. Click **Run pipeline**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `RESEARCHING` within 1 s more.
- Within ~25 s the card reaches `RESEARCHED`. The right pane shows the Market Research panel with ≥ 4 data point rows; each row has a non-empty sector, metric, value, and source.
- Within ~25 s more the card reaches `STRATEGIZED`. The right pane shows ≥ 2 allocation targets summing to 100% and ≥ 1 rationale item.
- Within ~25 s more the card reaches `PLANNED`. The right pane shows the Execution Plan with ≥ 2 sequenced action items, each with at least one instrument.
- Within ~25 s more the card reaches `ASSESSING`, then `EVALUATED` within a few seconds of that. The right pane shows the full Report with a risk band badge, 2 mitigations, and a compliance score chip showing 5/5.
- The disclaimer-log strip shows ≥ 4 entries, one per phase response.
- Total elapsed time: ≤ 90 s on the happy path.

## J2 — Disclaimer guardrail fires on every phase response

**Preconditions:** Service running (any model provider). Any seeded query submitted.

**Steps:**
1. Submit any seeded query.
2. Wait for `EVALUATED`.
3. Select the advisory in the live list and inspect the disclaimer-log strip on the right pane.

**Expected:**
- The disclaimer-log strip is visible and non-empty.
- It contains exactly 4 entries (one per phase: RESEARCH, STRATEGY, EXECUTE, ASSESS).
- Each entry shows an `injectedAt` timestamp and the first 80 characters of the disclaimer text: *"This content is for educational purposes only..."*
- The entity's `disclaimerLog` also has 4 entries when fetched via `GET /api/advisories/{id}`.
- No advisory reaches `EVALUATED` without a disclaimer-log of length 4. If the guardrail rejected even one injection, the advisory would be in `FAILED` state.

## J3 — Sector sanitizer redacts prohibited language

**Preconditions:** Mock LLM mode. The mock's `define-strategy.json` includes one entry whose `RationaleItem.text` contains the phrase `guaranteed return`.

**Steps:**
1. Submit any seeded query five times in a row.
2. Watch the live list for a card showing the small purple dot (sanitizer-event indicator).

**Expected:**
- On the affected advisory, the purple dot appears on the card in the live list.
- Selecting the advisory shows the sanitizer-event strip with one entry: `matchedPattern = "guaranteed"`, `redactedFragment` showing the original phrase, `firedAt` timestamp.
- The right pane's Strategy panel shows `[REDACTED — compliance]` in the `RationaleItem.text` field where the prohibited phrase was.
- The STRATEGY phase response written to `AdvisoryEntity` does NOT contain the word "guaranteed" — only the redacted marker.
- The advisory still reaches `EVALUATED`; the sanitizer is non-blocking.

## J4 — Low compliance score flags an advisory

**Preconditions:** Mock LLM mode. The mock's `assess-risk.json` includes one entry with `mitigations = []` and the mock's `define-strategy.json`'s paired entry has `allocations.size() = 1`.

**Steps:**
1. Submit any seeded query until the compliance score chip shows a value ≤ 2.
2. Observe the card border.

**Expected:**
- The compliance score chip on the card shows **2** (or **1** depending on which rules pass).
- The rationale reads: *"Risk mitigations absent and allocation breadth insufficient — only 1 target, 0 mitigations."*
- The card's border highlights red, drawing the reader's attention.
- The other advisories in the run scored ≥ 4.
- The advisory is complete (status `EVALUATED`); a low score does not stop the pipeline.

## J5 — Custom query with no matching market-data file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/market/<sector>.json` exists that matches the user's query topic.

**Steps:**
1. In the App UI, type a custom query (e.g. `cryptocurrency arbitrage strategy for dogecoin`) into the query field.
2. Click **Run pipeline**.

**Expected:**
- `ResearchTools.fetchMarketData` returns an empty list for the unrecognised sector.
- The agent's RESEARCH task returns a `MarketSnapshot` with `dataPoints = []`.
- The workflow advances to `strategyStep`. The DEFINE_STRATEGY task returns a `Strategy` with `approach = "(insufficient market data)"` and empty `allocations`.
- The workflow advances through `executionStep` and `riskStep`. Each returns minimal typed results consistent with empty upstream data.
- The compliance score chip shows 1 (no rules pass with empty data). The rationale names "no allocation targets, no risk mitigations."
- The pipeline completes; nothing crashes; the empty advisory is honestly minimal.
- Disclaimer injection still fires on every phase (4 entries in the disclaimer log).

## J6 — Partial-failure preserves completed phase data

**Preconditions:** Service running with a real model provider. A network interruption is simulated at the `executionStep` (e.g., by returning an error from the mock at the PLAN task).

**Steps:**
1. Configure the mock to return an error on the third task call (`PLAN_EXECUTION`).
2. Submit a seeded query and wait.

**Expected:**
- The advisory card transitions through `RESEARCHED` and `STRATEGIZED`, then stalls.
- After the `executionStep` exhausts its 2-retry budget, the workflow transitions to the error step.
- The entity records `AdvisoryFailed` with a reason string.
- The card shows status `FAILED` with a red status pill.
- The right pane still shows the completed Market Research panel and the completed Strategy panel — partial data from the two successful phases is preserved.
- The Execution Plan panel shows an empty or spinner state; it never shows corrupted data.
