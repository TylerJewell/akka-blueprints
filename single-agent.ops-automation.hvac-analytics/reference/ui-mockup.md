# UI mockup — hvac-analytics

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: HVAC Analytics</title>`.

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
- Headline: `HVAC <span class="accent">Analytics</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a zone (Zone-A, Zone-B, or Zone-C) and type or load a seeded question.
  3. Click **Ask**.
  4. Watch the card transition through SNAPSHOT_READY → ANALYSING → ANSWER_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → snapshot → analyse → eval; one paragraph on the performance-monitor evaluator.
- Card **Components**: rows per component (QueryEntity, TelemetryStore, QueryWorkflow, HvacAnalyticsAgent, AnswerQualityScorer, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Scorer as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `sector = building-operations` and `decisions.authority_level = recommend-only` are the prominent answers. `oversight.human_in_loop = true` is clearly marked. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (E1). ID badge coloured blue (eval-periodic).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.
- Below the table: a rolling-average chart showing the quality score trend across the last 20 queries submitted during the session (computed client-side from the SSE stream).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the answer.</span>`. Subtitle: `One agent, one quality evaluator on every answer.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Zone` (Zone-A / Zone-B / Zone-C / custom), `Question` textarea (with 6 seeded-question links that fill the textarea), `Submitted by` text input, and a yellow `Ask` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, trend badge (when answer landed), quality score chip (when eval landed), truncated question text, age.
  - **Right column** — Selected-query detail.
    - Header: status pill + trend badge + quality score chip + question text.
    - Telemetry snapshot: a compact table of metric, value, unit, zone, recordedAt for each point in the snapshot.
    - Assessment: the agent's 1–3-sentence paragraph.
    - Supporting data points: a table with metric, value, unit, zone, recordedAt, significance.
    - Trend badge: IMPROVING=green, STABLE=muted, DEGRADING=red, UNKNOWN=muted.
    - Recommended action: a highlighted text block.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: INITIATED=muted, SNAPSHOT_READY=blue, ANALYSING=yellow, ANSWER_RECORDED=blue, EVALUATED=green, FAILED=red.
- Trend badge colours: IMPROVING=green, STABLE=muted, DEGRADING=red, UNKNOWN=muted.
- Quality score chip colours: score 4–5=green, score 3=yellow, score 1–2=red.
