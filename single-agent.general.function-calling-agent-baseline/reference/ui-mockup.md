# UI mockup — function-calling-agent-baseline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Function Calling Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Function <span class="accent">Calling Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded query (arithmetic word problem / vocabulary / date calculation) or type your own.
  3. Select which tools to expose for this run (calculator, dictionary, date-formatter).
  4. Click **Run query** and watch the card transition through RUNNING → ANSWER_RECORDED with the tool-call trace appearing.
- Card **How it works**: one paragraph on submit → workflow start → agent tool loop → answer; one paragraph on the two governance guardrails (before-tool-call and before-agent-response).
- Card **Components**: rows per component (AgentRunEntity, AgentRunWorkflow, FunctionCallingAgent, ToolCallGuardrail, AnswerGuardrail, InProcessToolExecutor, AgentRunView, AgentRunEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, 2 Guardrails, 1 ToolExecutor).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = authoritative` and `oversight.human_in_loop = false` declarations are filled and prominent — they mark this as a baseline pattern that deployers may harden. Fields carrying `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, G2). ID badges coloured: G1 red (guardrail · before-tool-call), G2 red (guardrail · before-agent-response).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a query. <span class="accent">See the tool calls.</span>`. Subtitle: `One agent, two guardrails, three in-process tools.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded queries` (arithmetic / vocabulary / date-calculation / custom), `Query` textarea (with a "Load seeded example" link that fills it), a **Tool set** checkbox group (calculator, dictionary, date-formatter — all checked by default), `Submitted by` text input, and a yellow `Run query` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, guardrail badge (BLOCKED\_TOOL\_CALL / BLOCKED\_ANSWER / none), iteration count, query snippet, age.
  - **Right column** — Selected-run detail.
    - Header: status pill + guardrail badge + `query` (first 80 chars).
    - Enabled tools list: pill for each tool name.
    - Tool-call trace: collapsible per iteration. Each entry shows iteration number, tool name chip, arguments (key→value grid), result string.
    - Final answer section: the `answer` string in a readable paragraph block.
    - Guardrail summary at bottom: two rows (tool-call / answer) with blocked badge and block count.
- Status pill colours: SUBMITTED=muted, RUNNING=yellow, ANSWER_RECORDED=green, FAILED=red.
- Guardrail badge colours: none=muted, BLOCKED\_TOOL\_CALL=orange, BLOCKED\_ANSWER=red.
- Tool name chips: calculator=teal, dictionary=blue, date-formatter=purple.
