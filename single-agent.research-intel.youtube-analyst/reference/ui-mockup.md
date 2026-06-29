# UI mockup — youtube-analyst

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: YouTubeAnalyst</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The exemplar's `static-resources/index.html` has the canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

And the DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `YouTube<span class="accent">Analyst</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded channel fixture (consumer-electronics / developer-tools / lifestyle vlogger) and a period.
  3. Click **Run analysis**.
  4. Watch the card transition through METRICS_FETCHED → ANALYSING → REPORT_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on request → fetch metrics → analyse → eval; one paragraph on the two governance mechanisms (guardrail, on-decision eval).
- Card **Components**: rows per component (ChannelEntity, MetricsFetcher, AnalysisWorkflow, ChannelAnalystAgent, ReportGuardrail, AnalysisEvaluator, AnalysisView, AnalysisEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled and prominent — this blueprint handles public channel metrics, not personal data. `decisions.authority_level = inform-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Request an analysis. <span class="accent">Read the report.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Request panel + live list.
    - Request panel: dropdown `Channel` (consumer-electronics / developer-tools / lifestyle vlogger / custom), radio buttons or dropdown `Period` (Last 28 days / Last 90 days / All-time), radio buttons or dropdown `Focus area` (Growth / Engagement / Audience / All), `Requested by` text input, and a yellow `Run analysis` button.
    - Live list below: one card per analysis, newest-first. Each card shows status pill, tier badge (when report landed), eval score chip (when eval landed), channel handle, period, age.
  - **Right column** — Selected-analysis detail.
    - Header: status pill + tier badge + eval score chip + channel handle + period.
    - Metrics summary: subscriber count, subscriber delta (with arrow), total views, views delta, video count — rendered as a compact stat row.
    - Top-videos table: columns title, views, engagement rate (coloured chip), trend arrow.
    - Narrative summary: the agent's 2–4-sentence paragraph.
    - Recommendations list: each entry shows the `metricRef` chip and the `action` text.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: REQUESTED=muted, METRICS_FETCHED=blue, ANALYSING=yellow, REPORT_RECORDED=blue, EVALUATED=green, FAILED=red.
- Tier badge colours: GROWING=green, STEADY=muted, DECLINING=red.
- Trend arrow colours: UP=green, FLAT=muted, DOWN=red.
- Engagement-rate chip: ≥8%=green, 4–8%=blue, <4%=muted.
