# UI mockup — weather-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: WeatherAgent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Weather<span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Type a weather question or pick one of the four seeded examples.
  3. Select a unit system and click **Ask**.
  4. Watch the card transition through PROCESSING → ANSWERED (or FAILED) and see the tool-call timeline.
- Card **How it works**: one paragraph on submit → process → answer; one paragraph on the before-tool-call guardrail.
- Card **Components**: rows per component (QueryEntity, QueryWorkflow, WeatherAgent, ToolCallGuardrail, WeatherToolProvider, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 1 ToolProvider as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = informational` and `oversight.human_in_loop = false` declarations are the distinctive answers for this use case. `pii: false` in Data is noted. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- Row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Get the weather.</span>`. Subtitle: `One agent, one guardrail, two tools.`
- Layout: two-column.
  - **Left column** — Question panel + live list.
    - Question panel: `Question` textarea (placeholder: "What is the weather in Tokyo right now?"), unit-system radio buttons (Metric / Imperial / Standard), `Asked by` text input (optional), and a blue `Ask` button. Four seeded-example buttons below the textarea that fill in the question text with one click.
    - Live list below: one card per query, newest-first. Each card shows status pill, question excerpt, age.
  - **Right column** — Selected-query detail.
    - Header: status pill + question text + unit system badge.
    - Tool-call timeline: a vertical list of `ToolCallRecord` entries, each showing tool name, parameter chips, outcome badge (accepted / rejected / success / error), and relative timestamp.
    - Weather answer block (when `status = ANSWERED`):
      - Location header: display name, coordinates.
      - Current conditions card: temperature, feels-like, wind speed, humidity, description, icon.
      - Forecast table (when present): one row per `ForecastDay` with date, high/low, precipitation, description.
      - Narrative paragraph from `answer.narrative`.
    - Failure reason (when `status = FAILED`): a red callout with the `failureReason` string.
- Status pill colours: SUBMITTED=muted, PROCESSING=yellow, ANSWERED=green, FAILED=red.
- Outcome badge colours: accepted=muted, rejected=red, success=green, error=red.
