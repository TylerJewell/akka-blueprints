# UI mockup — structured-mcp-tool

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Structured Output Tool (MCP)</title>`.

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
- Headline: `Structured MCP <span class="accent">Tool</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded location (London / Tokyo / Sydney) or type your own.
  3. Click **Get weather**.
  4. Watch the card transition through RUNNING → COMPLETED and the result pane populate.
- Card **How it works**: one paragraph on submit → agent → MCP tool call → guardrail → summary; one paragraph on the `after-tool-call` guardrail mechanism.
- Card **Components**: rows per component (QueryEntity, WeatherQueryWorkflow, WeatherQueryAgent, McpToolResultGuardrail, InlineMcpServer, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 1 MCP Server as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `sector: general` declaration is prominent. `decisions.authority_level = informational-only` and `oversight.human_in_loop = false` are the distinctive answers for this domain. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a location. <span class="accent">Get the weather.</span>`. Subtitle: `One agent, one MCP tool, one guardrail.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: `Location` text input with seeded-location quick-pick buttons (London / Tokyo / Sydney), `Unit` toggle (Celsius / Fahrenheit), `Submitted by` text input, and a yellow `Get weather` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, conditions badge (when completed), location name, age.
  - **Right column** — Selected-query detail.
    - Header: status pill + conditions badge + location name.
    - Weather tiles row: temperature chip, humidity chip, wind speed chip.
    - Narrative paragraph: the agent's one-sentence description.
    - Unit note: if FAHRENHEIT was requested, show both values (e.g., "82.1 °F / 27.8 °C").
- Status pill colours: SUBMITTED=muted, RUNNING=yellow, COMPLETED=green, FAILED=red.
- Conditions badge colours: CLEAR=yellow, PARTLY_CLOUDY=blue, OVERCAST=muted, RAIN=blue, THUNDERSTORM=red, SNOW=light-blue, FOG=muted.
- Empty state in right column: "Select a query from the list to see its result."

The narrative paragraph is the agent's output — the only field in the UI that comes directly from the LLM. All numeric tiles come from the structured `WeatherSummary` fields, ensuring that even if the narrative were garbled, the structured data is independently readable.
