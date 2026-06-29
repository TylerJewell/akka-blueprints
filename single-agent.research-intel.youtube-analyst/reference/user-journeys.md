# User journeys — youtube-analyst

## J1 — Request a channel analysis and get a report

**Preconditions:** Service running on declared port (`http://localhost:9308/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9308/` → App UI tab.
2. From the **Channel** dropdown, pick `developer-tools creator`.
3. Set **Period** to `Last 28 days` and **Focus area** to `Engagement`.
4. Click **Run analysis**.

**Expected:**
- The new card appears in the live list with status `REQUESTED` within 1 s.
- The card transitions to `METRICS_FETCHED` within 1 s. The right-pane detail shows a metrics summary with subscriber count, subscriber delta, and the top-videos table (at minimum 3 rows, each with a non-zero engagement rate).
- Within 30 s the card reaches `REPORT_RECORDED`. The right pane shows: a tier badge (`GROWING`, `STEADY`, or `DECLINING`), the narrative summary paragraph containing at least one numeric figure, the top-videos table, and a non-empty recommendations list.
- Within 1 s of `REPORT_RECORDED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a malformed report

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `analyse-channel.json` includes deliberately malformed entries (a `topVideos` entry whose `videoId` is not in the fixture; a `recommendations` entry whose `metricRef` is `"undefined"`).

**Steps:**
1. Run three separate analyses on any seeded channel (J1 steps × 3, same or different channels).
2. Watch the third analysis's lifecycle in the network panel of the browser dev tools (`/api/analyses/sse`).

**Expected:**
- The third analysis's first agent iteration produces a malformed report.
- The `before-agent-response` guardrail rejects it. The malformed report NEVER lands in `ChannelEntity` — there is no `ReportRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed report. The card transitions to `REPORT_RECORDED` with a report that satisfies all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — Evidence-thin report flags eval score 1

**Preconditions:** Mock LLM mode. The mock has an entry that returns `TopVideo` records with `engagementRate = 0.0` for all videos.

**Steps:**
1. Run an analysis that causes the mock to select the evidence-thin entry (consult `src/main/resources/mock-responses/analyse-channel.json` for the entry's position in the mock rotation).
2. Wait for `EVALUATED`.

**Expected:**
- The report lands well-formed (the guardrail only checks structural validity and data consistency, not whether engagement rates are non-zero — that is the evaluator's domain).
- The eval score chip shows **1** and the rationale reads something like "TopVideo entries have zero engagement rate; report is not grounded in the metrics data."
- The card's border highlights red. The strategist knows to inspect this report before acting on it.

## J4 — Guardrail catches a metric-reference violation

**Preconditions:** Mock LLM mode. The mock has an entry that returns a `Recommendation` with `metricRef = "undefined"`.

**Steps:**
1. Trigger the third analysis in the rotation (J2 steps apply).
2. Observe the SSE stream for the `ANALYSING` state and confirm the card eventually reaches `REPORT_RECORDED`.

**Expected:**
- The first agent iteration produces a report with the invalid `metricRef`.
- The guardrail rejects it with error code `invalid-metric-ref`.
- The second iteration produces a report where all `metricRef` values are recognized field names.
- The UI card shows `REPORT_RECORDED` with a valid report. The invalid-reference version never appears in any API response.

## J5 — All three seeded fixtures produce distinct tier classifications

**Preconditions:** Service running with any model provider (the tier classification uses deterministic fixture data via the `AnalysisEvaluator`'s consistency check).

**Steps:**
1. Run an `ALL`-focus analysis on each of the three seeded channels: consumer-electronics, developer-tools, lifestyle vlogger.
2. Wait for all three to reach `EVALUATED`.

**Expected:**
- The consumer-electronics fixture produces `STEADY` (flat subscriber delta, high absolute views).
- The developer-tools fixture produces `GROWING` (positive subscriber delta over 28 days).
- The lifestyle vlogger fixture produces `DECLINING` (negative subscriber delta in the default period).
- No two fixture analyses share the same tier badge in the live list.
- All three analyses receive eval scores ≥ 3 (the seeded fixture data is intentionally designed to produce grounded reports).
