# UI mockup — any-llm-tool-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Agent From Any LLM</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Agent From <span class="accent">Any LLM</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the three seeded queries (Tokyo / London / São Paulo) or type your own.
  3. Pick a backend from the dropdown (or leave it on `mock`).
  4. Click **Ask** and watch the card transition through INVOKING → RESULT_RECORDED.
- Card **How it works**: one paragraph on submit → invoke → tool-call → report; one paragraph on the `before-tool-call` guardrail protecting the tool boundary.
- Card **Components**: rows per component (QueryEntity, QueryWorkflow, WeatherAgent, WeatherToolGuardrail, WeatherTools, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 1 Tool class as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `decisions.authority_level = informational` and `oversight.human_in_loop = false` answers are prominent. The absence of PII handling (`pii: false`) is explicitly surfaced. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- Row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask about the weather. <span class="accent">Watch it run.</span>`. Subtitle: `One agent, one tool, one guardrail.`
- Layout: two-column.
  - **Left column** — Query submission panel + live list.
    - Submission panel: **Backend** dropdown (anthropic / openai / googleai-gemini / ollama / mock), **Query** textarea (with three "Load example" links for Tokyo / London / São Paulo), **Submitted by** text input, and a yellow **Ask** button.
    - Live list below: one card per query, newest-first. Each card shows status pill, condition badge (when report landed), query text (truncated to 60 chars), backend badge, age.
  - **Right column** — Selected-query detail.
    - Header: status pill + backend badge + query text.
    - Weather report panel (visible once `RESULT_RECORDED`):
      - Location header.
      - Condition badge (large).
      - Three stat tiles: Temperature (°C), Humidity (%), Wind (km/h).
      - Narrative sentence in italic.
    - Status panel (visible while `INVOKING`): a spinner with the text "Agent is calling get_weather…".
    - Error panel (visible on `FAILED`): red border, error reason text.
- Status pill colours: SUBMITTED=muted, INVOKING=yellow, RESULT_RECORDED=green, FAILED=red.
- Condition badge colours: SUNNY=yellow, CLOUDY=muted, RAINY=blue, SNOWY=light-blue, UNKNOWN=grey.
- Backend badge colours: anthropic=purple, openai=green, googleai-gemini=blue, ollama=orange, mock=muted.

The `requestedBackend` field is for illustration and traceability. The UI lets the user pick it, but the backend actually used depends on which model-provider blocks are configured in `application.conf`. If the requested backend's block is absent, the agent falls back to the configured default.
