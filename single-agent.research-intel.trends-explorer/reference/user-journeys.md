# User journeys — google-trends-agent

## J1 — Submit a US / Past week / Technology query and get a report

**Preconditions:** Service running on declared port (`http://localhost:9674/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9674/` → App UI tab.
2. Select **Region**: US. Select **Time window**: Past week. Select **Category**: Technology.
3. Enter any value in **Submitted by** (e.g., `analyst-1`).
4. Click **Fetch trends**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `DATA_READY` within 1 s. The right-pane detail shows the raw payload summary: topic count ≥ 8, breakout-topic count ≥ 2, fetched-at timestamp.
- Within 30 s the card reaches `REPORT_READY`. The right pane shows a ranked topic table with one row per topic in the payload. Every row has a non-empty rationale, a search volume index bar chip, and a breakout flag where applicable.
- The breakout signal paragraph appears in the highlighted block, naming at minimum the breakout topics found and why they registered.
- Within 1 s of `REPORT_READY`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a malformed report

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `synthesize-trends.json` includes deliberately malformed entries (one with an empty `rationale` on a topic; one with duplicate rank values).

**Steps:**
1. Submit three requests in a row with the same region/window/category (J1 steps × 3).
2. Watch the third submission's lifecycle in the browser dev tools network panel (`/api/trends/sse`).

**Expected:**
- The third request's first agent iteration produces a malformed report.
- The `before-agent-response` guardrail rejects it. The malformed report NEVER lands in `TrendRequestEntity` — there is no `ReportRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed report. The card transitions to `REPORT_READY` with a report satisfying all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — Evidence-thin report flags eval score 1

**Preconditions:** Mock LLM mode. A specific mock-response entry is configured to return topics with empty `rationale` strings.

**Steps:**
1. Submit a request whose `requestId` resolves (via `MockModelProvider.seedFor`) to the "evidence-thin" mock entry (empty rationale on all topics).
2. Wait for `EVALUATED`.

**Expected:**
- The report lands well-formed (the guardrail only checks structural validity — empty rationale strings would be caught on a retry, but this path simulates the guardrail accepting them for the eval path to exercise).
- The eval score chip shows **1** and the rationale reads something like: "Topic rationale strings are empty; report is not anchored in the data."
- The card's border highlights red. The analyst knows to treat this report with caution.

## J4 — Related clusters match topic names

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a request for APAC / Past month / Finance (a seeded dataset with 10 topics, each with related queries).
2. Wait for `EVALUATED`.
3. Open the Related clusters accordion in the right pane.

**Expected:**
- The accordion shows one entry per topic that has related queries. Each entry's `anchorTerm` exactly matches a `topicName` value from the ranked topic table above.
- No cluster has an `anchorTerm` that is absent from the ranked topic list.
- The eval score is ≥ 4 (all anchors matched, rationales are non-empty, breakout summary is present).

## J5 — Full topic coverage — all payload topics appear in the report

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a request for EU / Past week / All (the "All" category seed has 12 topics).
2. Wait for `REPORT_READY`.

**Expected:**
- The ranked topic table has exactly 12 rows — one per topic in the seeded payload.
- Rank values run strictly 1 through 12 with no gaps and no duplicates.
- Topics are ordered by `searchVolumeIndex` descending in the table; any ties are broken alphabetically by `topicName`.
- If the agent's first iteration had duplicate or missing ranks, the guardrail would have rejected it — the absence of a guardrail rejection in the log confirms the first iteration was rank-complete.
