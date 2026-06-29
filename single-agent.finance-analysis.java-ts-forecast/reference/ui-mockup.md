# UI mockup — time-series-forecasting (java)

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: TSForecast</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `TS<span class="accent">Forecast</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded series (Revenue / Inventory Demand / Credit Loss) and load both the series data and the default configuration with one click.
  3. Click **Submit for forecast**.
  4. Watch the card transition through VALIDATED → FORECASTING → FORECAST_COMPLETED → DRIFT_EVALUATED.
- Card **How it works**: one paragraph on submit → validate → forecast → drift-eval; one paragraph on the two governance mechanisms (guardrail, drift-fairness-watch).
- Card **Components**: rows per component (ForecastEntity, SeriesValidator, ForecastWorkflow, ForecastingAgent, ForecastGuardrail, DriftEvaluator, ForecastView, ForecastEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `financial_time_series: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-periodic).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a series. <span class="accent">Read the forecast.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Series` (Revenue / Inventory Demand / Credit Loss / custom), `Series name` text input, `Historical data` textarea (CSV, date + value columns, with a "Load seeded example" link that fills both name and data), `Horizon (days)` select (7 / 14 / 30), `Confidence level` select (80% / 95%), `Granularity` select (Daily / Weekly), `Submitted by` text input, and a yellow `Submit for forecast` button.
    - Live list below: one card per forecast, newest-first. Each card shows status pill, drift badge (when drift eval landed), series name, age.
  - **Right column** — Selected-forecast detail.
    - Header: status pill + drift badge + `seriesName`.
    - Series quality summary: row count, date range, missing-value count, stationarity chip.
    - Forecast table: columns date, point forecast, lower bound, upper bound, step quality score (coloured 1–5). Scrollable if horizon > 14.
    - Summary paragraph: the agent's 2–4 sentence description of trend and confidence.
    - Drift eval section at bottom: MAPE value, drift status badge (STABLE=green, WATCH=yellow, ALERT=red), one-line rationale. ALERT highlights the card border red.
- Status pill colours: SUBMITTED=muted, VALIDATED=blue, FORECASTING=yellow, FORECAST_COMPLETED=blue, DRIFT_EVALUATED=green, FAILED=red.
- Drift badge colours: STABLE=green, WATCH=yellow, ALERT=red.
- Step quality score chip colours: 5=green, 4=green-muted, 3=blue, 2=yellow, 1=red.

The raw historical series is not displayed in the detail pane — only the quality summary and the forecast result. Analysts who need the raw data fetch `/api/forecasts/{id}` and read `submission.historicalData` from the JSON. This is intentional: the UI demonstrates what the model consumed (quality-checked series metadata) rather than replaying the full input.
