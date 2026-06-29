# UI mockup — tool-search-discovery

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Deferred Tool-Discovery Harness</title>`.

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
- Headline: `Deferred Tool-<span class="accent">Discovery Harness</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded task query (weather / file-ops / text-pipeline) or type your own.
  3. Click **Submit task**.
  4. Watch the card transition through DISCOVERING → READY → EXECUTING → COMPLETED and check the discovered-tools list alongside the result.
- Card **How it works**: one paragraph on submit → discover → guard → execute → result; one paragraph on the allowlist guardrail mechanism.
- Card **Components**: rows per component (TaskEntity, ToolCatalogConsumer, ToolRegistry, DiscoveryWorkflow, DiscoveryAgent, ToolAllowlistGuardrail, TaskView, TaskEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Registry as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without these, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. `decisions.authority_level = full-automation` and `oversight.human_in_loop = false` (with `human_on_loop = true`) are the distinctive answers. `TO_BE_COMPLETED_BY_DEPLOYER` fields are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">See the tools used.</span>`. Subtitle: `One agent, one allowlist guardrail.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: `Task query` textarea with three seed-query links ("Load weather example", "Load file-ops example", "Load text-pipeline example"), `Submitted by` text input, and a yellow `Submit task` button.
    - Live list below: one card per task, newest-first. Each card shows status pill, age, and a short excerpt of the user query.
  - **Right column** — Selected-task detail.
    - Header: status pill + taskId excerpt + age.
    - Discovered tools: a chip list of tool names found by the registry search. Each chip shows the tool's category colour.
    - Guarded tools: a chip list (red) of tool ids the allowlist blocked, or a muted "none blocked" label.
    - Result output: a monospace block with the agent's `output` text.
    - Tools used: a chip list (green) of tool ids the agent successfully called.
- Status pill colours: RECEIVED=muted, DISCOVERING=blue, READY=blue, EXECUTING=yellow, COMPLETED=green, FAILED=red.
- Tool category chip colours: weather=sky-blue, file-ops=amber, text=violet.
- Guarded-tools chips always red; tools-used chips always green.

The catalog attachment the agent received is not displayed on this screen. A developer inspecting what the agent saw can fetch `GET /api/tasks/{id}` and read `discovered.tools[].schemaJson` from the JSON.
