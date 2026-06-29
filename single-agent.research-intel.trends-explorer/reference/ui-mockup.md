# UI mockup — google-trends-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Google Trends Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. Canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Google Trends <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a region, time window, and category from the dropdowns (or use the defaults: US / Past week / Technology).
  3. Click **Fetch trends**.
  4. Watch the card transition through DATA_READY → SYNTHESIZING → REPORT_READY → EVALUATED.
- Card **How it works**: one paragraph on submit → data-fetch → synthesize → eval; one paragraph noting that the on-decision evaluator is deterministic (no LLM call) and scores every report for coverage quality.
- Card **Components**: rows per component (TrendRequestEntity, TrendDataFetcher, TrendRequestWorkflow, TrendsExplorerAgent, ReportQualityGuardrail, ReportEvaluationScorer, TrendReportView, TrendEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is visible and prominent. `decisions.authority_level = inform-only` and `oversight.human_on_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Empty state: a centred message reading "This baseline registers no enforcement controls. Add controls in `eval-matrix.yaml` after wiring your own governance mechanisms."
- No rows, no table header. The empty-state card explains what controls are and links to the eval-matrix schema documentation.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a query. <span class="accent">Read the trends.</span>`. Subtitle: `One agent, seeded data, deterministic quality scoring.`
- Layout: two-column.
  - **Left column** — Query panel + live list.
    - Query panel: `Region` dropdown (US, EU, APAC, LATAM, Global; plus a text input for custom BCP-47 code), `Time window` dropdown (Past day, Past week, Past month, Past 90 days), `Category` dropdown (All, Technology, Finance, Health, Entertainment; plus a custom text input), `Submitted by` text input, and a yellow `Fetch trends` button.
    - Live list below: one card per request, newest-first. Each card shows status pill, region + window label (e.g. "US · Past week"), category badge, eval score chip (when available), and age.
  - **Right column** — Selected-request detail.
    - Header: status pill + eval score chip + region/window/category label.
    - Query parameters: a small grid showing region, time window, category, submitted by, submitted at.
    - Raw payload summary: topic count, breakout-topic count, fetched-at timestamp (the full payload body is available via `GET /api/trends/{id}` — not displayed inline).
    - Ranked topic table: columns rank (#), topic name, search volume index (bar chip 0–100), breakout flag (lightning bolt if true), rationale.
    - Breakout signal paragraph: the agent's `breakoutSummary` in a highlighted block.
    - Related clusters accordion: one row per cluster, expand to see the up-to-3 related queries.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, DATA_READY=blue, SYNTHESIZING=yellow, REPORT_READY=blue, EVALUATED=green, FAILED=red.
- Breakout flag: a lightning bolt icon (⚡) in yellow when `breakout=true`, absent when false.
- Search volume bar chip: a short horizontal bar from 0 to 100 rendered inline in the rank column cell.
