# UI mockup — multi-tool-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: MultiToolAgent</title>`.

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
- Headline: `Multi<span class="accent">Tool</span>Agent`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded request example or type your own multi-part question.
  3. Click **Submit**.
  4. Watch the tool-call log fill in and the answer appear as the session reaches ANSWERED.
- Card **How it works**: one paragraph on submit → validate → dispatch → answer; one paragraph on the `before-tool-call` guardrail and how it validates inputs before each tool fires.
- Card **Components**: rows per component (SessionEntity, SessionWorkflow, ToolCallingAgent, ToolCallGuardrail, WeatherTool, CurrencyTool, UnitTool, SessionView, SessionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, 3 Tool adapters, 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `external_tool_calls` list (weather, currency, unit) in Operations is filled and prominent. `decisions.authority_level = answer-only` and `oversight.human_in_loop = false` are the distinctive answers. Deployer-specific fields are `TO_BE_COMPLETED_BY_DEPLOYER` and displayed faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Watch the tools work.</span>`. Subtitle: `One agent, three tools, one guardrail between them and the network.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a `Request` textarea (with five seeded-example chips below it — click to auto-fill), a `Submitted by` text input, and a yellow `Submit` button.
    - Live list below: one card per session, newest-first. Each card shows status pill, session age, and the first 80 characters of the request text.
  - **Right column** — Selected-session detail.
    - Header: status pill + `sessionId` (truncated).
    - Request text: the full question in a monospace block.
    - Tool-call log: a table with columns — tool name (coloured chip), input args (key=value pairs), result (truncated to 120 chars, full on hover), guardrail rejection (red text if present), timestamp. Rows appear incrementally as `ToolCallRecorded` SSE events arrive.
    - Answer section: the agent's combined answer paragraph, appears when status reaches ANSWERED.
    - FAILED state: shows a red banner with the failure reason and the partial tool-call log.
- Status pill colours: SUBMITTED=muted, DISPATCHING=yellow, ANSWERED=green, FAILED=red.
- Tool chip colours: WeatherTool=blue, CurrencyTool=purple, UnitTool=orange.
- Guardrail rejection rows are highlighted with a red left border.
