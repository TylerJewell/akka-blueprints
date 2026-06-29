# UI mockup — akka-bridge

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Agents</title>`.

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
- Headline: `Akka <span class="accent">Agents</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded tool set (search-tools / data-tools / math-tools) and load the matching seed task, or type your own.
  3. Click **Submit run**.
  4. Watch the tool-call log fill in as the agent works; the run card transitions through RUNNING → COMPLETED (or BLOCKED).
- Card **How it works**: one paragraph on submit → run → tool-call-guardrail → adapter → result; one paragraph on the governance mechanism (before-tool-call guardrail: permitted-tools, schema, budget).
- Card **Components**: rows per component (RunEntity, AgentFrameworkAdapter, RunWorkflow, BridgeAgent, ToolCallGuardrail, MockToolExecutor, RunView, RunEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Executor as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `authority_level: fully-autonomous` declaration in Decisions is filled and prominent. `oversight.human_on_loop = true` (audit-log monitoring) is the distinctive oversight posture. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Watch the tools run.</span>`. Subtitle: `One agent. Every tool call governed.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Tool set` (search-tools / data-tools / math-tools / custom), `Task description` textarea (with a "Load seeded example" link that fills the textarea and the tool-set dropdown), `Tool budget` number input (default 4), `Submitted by` text input, and a yellow `Submit run` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, age, and the task description truncated to 60 chars.
  - **Right column** — Selected-run detail.
    - Header: status pill + `taskDescription` (full text).
    - Permitted tools: a compact list of tool name chips with their descriptions on hover.
    - Budget bar: `N / toolBudget` permitted calls used, displayed as a fill bar. Turns red when budget is exhausted.
    - Tool-call log: a table with columns call #, tool name, guardrail verdict chip (PERMITTED=green / BLOCKED=red), argument snippet (first 80 chars of `argumentsJson`), result snippet (first 80 chars of `resultJson`), and timestamp.
    - Final answer block: the `result.answer` string in a styled box. Shown only when status is `COMPLETED` or `BLOCKED`.
- Status pill colours: ACCEPTED=muted, RUNNING=yellow, COMPLETED=green, BLOCKED=red, FAILED=red.
- Guardrail verdict chip colours: PERMITTED=green, BLOCKED=red.

The full `argumentsJson` and `resultJson` for each tool call are available via `GET /api/runs/{id}` — the UI shows only snippets to keep the log scannable.
