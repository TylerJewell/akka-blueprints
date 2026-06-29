# UI mockup — earth-engine-geospatial

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Earth Engine Geospatial Analyst</title>`.

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
- Headline: `Earth Engine <span class="accent">Geospatial Analyst</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded indicator set (Vegetation / Surface Temperature / Flood Extent) and enter a bounding region, or load a pre-filled example with one click.
  3. Click **Submit query**.
  4. Watch the card transition through DATASET_READY → ANALYSING → REPORT_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → validate → fetch → analyse → eval; one paragraph on the three governance mechanisms (bounds sanitizer, guardrail, on-decision eval).
- Card **Components**: rows per component (AnalysisEntity, DatasetFetcher, AnalysisWorkflow, GeospatialAnalystAgent, ReportGuardrail, EvaluationScorer, AnalysisView, AnalysisEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `location-fine-grained: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, E1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a query. <span class="accent">Read the report.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Query panel + live list.
    - Query panel: dropdown `Indicator set` (Vegetation / Surface Temperature / Flood Extent / custom), `Query label` text input, four coordinate inputs (`SW Lat`, `SW Lon`, `NE Lat`, `NE Lon`), `Start date` and `End date` date inputs, a "Load seeded example" link that fills all fields from the selected indicator set, `Submitted by` text input, and a yellow `Submit query` button.
    - Live list below: one card per analysis, newest-first. Each card shows status pill, report status badge (when report landed), eval score chip (when eval landed), query label, age.
  - **Right column** — Selected-analysis detail.
    - Header: status pill + report status badge + eval score chip + `queryLabel`.
    - Bounding region summary: SW/NE coordinate pairs + date range + computed area in km².
    - Submitted indicators: a small list with `indicatorId`, `datasetLayer` chip, and `unit`.
    - Dataset snapshot preview: a table with one row per layer (layer name, mean, min, max, missingDataPct%). Missing-data > 30% rows are flagged amber.
    - Report summary: the agent's 1–3-sentence paragraph.
    - Findings table: columns indicator id, severity (coloured chip), dataset layer, statistic (monospace), anomaly badge + confidence bar, recommendation.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, DATASET_READY=blue, ANALYSING=yellow, REPORT_RECORDED=blue, EVALUATED=green, FAILED=red.
- Report status badge colours: NOMINAL=green, ANOMALY_DETECTED=red, INSUFFICIENT_DATA=muted.
- Severity chip colours: INFO=muted, WATCH=blue, ALERT=yellow, CRITICAL=red.

The bounding region is displayed as numeric coordinates. Spatial visualisation (e.g., a map widget) is not included in this baseline — a deployer would add a map overlay using their chosen mapping library.
