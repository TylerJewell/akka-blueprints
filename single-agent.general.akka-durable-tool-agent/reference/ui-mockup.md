# UI mockup — durable-weather-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Durable Weather Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index:

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
- Headline: `Durable <span class="accent">Weather Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded city (Berlin / Tokyo / São Paulo / Nairobi) or type your own query.
  3. Pick an invocation mode (`trigger_agent` or `call_agent`) and click **Submit query**.
  4. Watch the job card transition through QUEUED → RUNNING → COMPLETED.
- Card **How it works**: one paragraph on submit → workflow start → agent tool calls → report assembly; one paragraph on the before-tool-call guardrail and the two invocation modes.
- Card **Components**: rows per component (WeatherJobEntity, AgentWorkflow, WeatherAgent, WeatherToolActivity, ToolCallGuardrail, JobView, JobEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 1 Activity as supporting constructs).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Component-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. `decisions.authority_level = inform-only` and `oversight.human_in_loop = false` are the distinctive answers. `invocation_modes: [trigger_agent, call_agent]` is shown in the Operations section. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a query. <span class="accent">Get a report.</span>`. Subtitle: `One agent, two invocation modes, one guardrail.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: `City / query` textarea (pre-populated with seeded city buttons: Berlin, Tokyo, São Paulo, Nairobi), `Invocation mode` radio group (`trigger_agent` / `call_agent`), `Submitted by` text input, and a yellow `Submit query` button.
    - Live list below: one card per job, newest-first. Each card shows status pill, mode badge (TRIGGER / CALL), city resolved (when available), age.
  - **Right column** — Selected-job detail.
    - Header: status pill + mode badge + resolved city + age.
    - Query text block.
    - Conditions card: temperature (large), condition label, humidity %, wind speed kph, observed-at timestamp.
    - Forecast table: columns date, high °C, low °C, condition, precipitation %.
    - Tool-call evidence table: columns tool name, arguments, status chip (EXECUTED / BLOCKED / FAILED), raw response snippet (truncated), guardrail rejection reason (shown only when status = BLOCKED).
    - Narrative summary paragraph.
- Status pill colours: QUEUED=muted, RUNNING=yellow, COMPLETED=green, FAILED=red.
- Mode badge colours: TRIGGER=blue, CALL=purple.
- Tool-call status chip colours: EXECUTED=green, BLOCKED=orange, FAILED=red.
- BLOCKED rows in the evidence table are highlighted with a faint orange row background.

The `call_agent` mode badge on a card links to the parent job card if a parent `jobId` is available in the entity. This gives a visual trace of the child-workflow composition path.
