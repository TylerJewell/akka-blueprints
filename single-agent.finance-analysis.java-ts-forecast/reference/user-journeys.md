# User journeys — time-series-forecasting (java)

## J1 — Submit a revenue series and get a 14-day forecast

**Preconditions:** Service running on declared port (`http://localhost:9643/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9643/` → App UI tab.
2. From the **Series** dropdown, pick `Revenue (90 days)`.
3. Click **Load seeded example** to fill the series name and historical-data textarea.
4. Set **Horizon** to `14`, **Confidence level** to `95%`, **Granularity** to `Daily`.
5. Click **Submit for forecast**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `VALIDATED` within 1 s. The right-pane detail shows the series quality summary: row count 90, date range, missing-value count 0, stationarity chip `STATIONARY`.
- Within 30 s the card reaches `FORECAST_COMPLETED`. The right pane shows: a 14-row forecast table with non-null `pointForecast`, `lowerBound < pointForecast < upperBound` for every row, and a `stepQualityScore` that decreases toward row 14. The summary paragraph describes trend direction and confidence level.
- Within 1 s of `FORECAST_COMPLETED`, the card reaches `DRIFT_EVALUATED` and shows the drift badge (STABLE / WATCH / ALERT) and the MAPE value.

## J2 — Guardrail blocks a malformed forecast result

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `forecast-series.json` includes deliberately malformed entries (one with `lowerBound > pointForecast`; one with `steps` count mismatching the requested horizon).

**Steps:**
1. Submit any seeded series three times in a row (J1 steps × 3), each time with a 14-day horizon.
2. Watch the third submission's lifecycle in the browser dev tools network panel (`/api/forecasts/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed result (interval inversion or step-count mismatch).
- The `before-agent-response` guardrail rejects it. The malformed result NEVER lands in `ForecastEntity` — there is no `ForecastCompleted` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed result. The card transitions to `FORECAST_COMPLETED` with a result that satisfies all three guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration, naming the failed check (e.g., `interval-inversion` or `step-count-mismatch`).

## J3 — High MAPE triggers ALERT drift status

**Preconditions:** Mock LLM mode. A specific mock entry produces forecast values that diverge significantly from the last actuals window (MAPE > 25%).

**Steps:**
1. Submit the `Credit Loss (365 days)` seeded series with a 30-day horizon.
2. If using the mock LLM, the `forecast-series.json` "high-drift" entry is selected for this series (by seedFor logic).
3. Wait for `DRIFT_EVALUATED`.

**Expected:**
- The drift badge shows `ALERT` (red).
- The MAPE value displayed is greater than 0.25 (e.g., `0.31`).
- The card border highlights red. The analyst knows to investigate the forecast before relying on it.
- The one-line rationale reads something like: "MAPE of 31.0% against the last 30-day actuals window exceeds the ALERT threshold; model calibration may need review."

## J4 — Invalid series triggers FAILED status

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the App UI, pick `custom` from the Series dropdown.
2. Enter a series name and paste a CSV where the `value` column contains non-numeric entries (e.g., `"N/A"`, `"—"`, `"pending"`).
3. Click **Submit for forecast**.

**Expected:**
- The card appears with status `SUBMITTED` briefly, then transitions to `FAILED`.
- The right-pane detail shows the failure reason: "value column contains non-numeric entries at rows [list]".
- No `ForecastWorkflow` is started for this forecast — `SeriesValidator` calls `ForecastEntity.fail(reason)` instead of `attachValidated`.
- No LLM call is made. The UI does not crash; the live list continues to display previous forecasts normally.

## J5 — Short series lowers step quality scores

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a custom series with exactly 8 data points and a 7-day horizon.
2. Wait for `DRIFT_EVALUATED`.

**Expected:**
- The `SeriesValidator` accepts the series (minimum is 14 for daily; however for 8-point series it still proceeds, marking `qualitySummary` as "Short series — confidence scores reduced").
- All 7 `HorizonStep.stepQualityScore` values are 1 or 2.
- The forecast summary paragraph notes "fewer than 14 data points; all step quality scores are 1."
- The drift eval completes without error (drift window is `min(14, rowCount)` = 8 actual points vs. 7 forecast steps, which is valid).
