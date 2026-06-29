# User journeys — earth-engine-geospatial

## J1 — Submit a vegetation query and get a report

**Preconditions:** Service running on declared port (`http://localhost:9104/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9104/` → App UI tab.
2. From the **Indicator set** dropdown, pick `Vegetation (5 indicators)`.
3. Click **Load seeded example** to fill all coordinate inputs, date fields, and the indicator list.
4. Click **Submit query**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `DATASET_READY` within 1 s. The right-pane detail shows the dataset snapshot table with one row per indicator; `pixelResolutionMeters` and `totalPixels` are displayed in the header.
- Within 30 s the card reaches `REPORT_RECORDED`. The right pane shows: a report status badge (NOMINAL / ANOMALY_DETECTED / INSUFFICIENT_DATA), the summary paragraph, and a finding row for *each of the 5 submitted indicators*. Every finding has a non-empty `datasetLayer`, a non-empty `statistic`, and a non-empty `recommendation`.
- Within 1 s of `REPORT_RECORDED`, the card reaches `EVALUATED` and shows an eval score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a malformed report

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `analyse-region.json` includes deliberately malformed entries (a finding whose `indicatorId` is not in the submitted list; a finding with `anomalyConfidence = 1.5`).

**Steps:**
1. Submit any seeded query three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/analyses/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed report.
- The `before-agent-response` guardrail rejects it. The malformed report NEVER lands in `AnalysisEntity` — there is no `ReportRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed report. The card transitions to `REPORT_RECORDED` with a report that satisfies all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — Evidence-thin report flags eval score 1

**Preconditions:** Mock LLM mode. The mock produces a report whose findings have non-empty `indicatorId` and `datasetLayer` but empty `statistic` strings.

**Steps:**
1. In the App UI, pick `Flood Extent (4 indicators)` and click **Load seeded example**.
2. Submit the query. (The mock selector picks the "evidence-thin" entry on a specific seed — see `analyse-region.json`.)

**Expected:**
- The report lands well-formed (the guardrail only checks structural validity, not evidence quality).
- The eval score chip shows **1** and the rationale reads "Findings cite layers but statistic strings are empty; report is not grounded in dataset values."
- The card's border highlights red. The researcher knows to inspect this report before acting.

## J4 — Oversized bounding box rejected before dataset assembly

**Preconditions:** Service running (real or mock LLM).

**Steps:**
1. In the App UI, enter SW Lat = -35, SW Lon = -80, NE Lat = 5, NE Lon = -34 (a region exceeding 500 000 km²).
2. Enter any query label and pick the Vegetation indicator set.
3. Click **Submit query**.

**Expected:**
- The card appears in the live list with status `SUBMITTED`, then transitions immediately to `FAILED` (within 1 s, before any dataset assembly begins).
- The right-pane detail shows the failure reason: "Bounding box area {N} km² exceeds the maximum permitted 500 000 km²."
- No `DatasetReady` event is emitted. No `AnalysisWorkflow` is started. No LLM call is made.
- The service log shows a single bounds-check rejection line with the computed area.

## J5 — Multi-indicator completeness

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a seeded example using the `Surface Temperature (4 indicators)` set.
2. Wait for `EVALUATED`.

**Expected:**
- The report's `findings` array has exactly 4 entries, one per submitted indicator. No silent omissions.
- Each `findings[i].indicatorId` matches one of the 4 submitted `indicatorId` values, one-to-one.
- The guardrail's absence from the log for this submission confirms the first agent iteration was structurally complete.

## J6 — Insufficient-data report when missing-data exceeds threshold

**Preconditions:** Mock LLM mode with a dataset fixture that sets `missingDataPct = 85` for more than half the layers.

**Steps:**
1. Submit the `Flood Extent (4 indicators)` seeded example over a date range where the "high-missing-data" fixture is selected by the mock seed.
2. Wait for `REPORT_RECORDED`.

**Expected:**
- The report's `status` is `INSUFFICIENT_DATA`.
- Each finding whose layer had `missingDataPct > 30` has `statistic` referencing the missing-data figure and a recommendation beginning with "Resubmit" or "Acquire".
- The eval score is ≥ 3 because the findings correctly cite the missing-data statistic — structurally sound even though the data quality is low.
