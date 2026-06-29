# UI mockup — computer-use-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: ComputerUseAgent</title>`.

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
- Headline: `Computer Use <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a task template (fill-web-form / navigate-and-extract / bulk-rename-files) or type a free-form task description.
  3. Click **Run task**.
  4. Watch the action log grow in real time; approve or reject high-impact actions from the confirmation banner; halt the task at any point.
- Card **How it works**: one paragraph on submit → start → action loop (observe → propose → guardrail → execute) → complete; one paragraph on the three governance mechanisms (before-tool-call guardrail, operator halt, human confirmation gate).
- Card **Components**: rows per component (TaskEntity, ConfirmationGateway, TaskExecutionWorkflow, ComputerUseAgent, ActionGuardrail, EnvironmentSimulator, TaskView, TaskEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Simulator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

Mermaid must include:

```js
mermaid.initialize({
  theme: 'dark',
  themeVariables: {
    nodeTextColor: '#e6edf3',
    stateLabelColor: '#e6edf3',
    transitionLabelColor: '#cccccc'
  }
});
```

And the CSS overrides:

```css
.statediagram-state text { fill: #e6edf3 !important; }
.edgeLabel foreignObject { overflow: visible !important; }
.edgeLabel { color: #cccccc !important; }
```

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `autonomous-action-with-hitl-gates` decisions surface and `operator_can_halt_at_any_time: true` are distinctive. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, C1). ID badges coloured: G1 red (guardrail), H1 orange (halt), C1 teal (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Watch it execute.</span>`. Subtitle: `One agent, three governance mechanisms around every action.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Task template` (fill-web-form / navigate-and-extract / bulk-rename-files / custom), `Task description` textarea (pre-filled from template selection; editable), `Submitted by` text input, and a yellow `Run task` button.
    - Live list below: one card per task, newest-first. Each card shows status pill, outcome badge (when complete), total actions / blocked count, task description (truncated), age. A red **Halt** button appears on cards in `RUNNING` or `AWAITING_CONFIRMATION` state.
  - **Right column** — Selected-task detail.
    - Header: status pill + outcome badge + `description` (truncated to one line).
    - Action log: a scrollable table of `ActionRecord` entries — columns: timestamp, action type (coloured chip), target selector (monospace, truncated), status chip (ALLOWED green / BLOCKED red / AWAITING_CONFIRMATION yellow / CONFIRMED teal / REJECTED orange), blocked reason (shown inline when status is BLOCKED).
    - Screenshot panel: renders the current screenshot from `EnvironmentSimulator`. Refreshes on each `action-executed` SSE sub-event.
    - Confirmation banner (shown only when `status == AWAITING_CONFIRMATION`): displays the pending action type, target, payload, and `riskRationale`. Two buttons: green **Approve** and red **Reject**. Clicking either POSTs to `/api/tasks/{id}/confirm`.
    - Outcome section (shown when `status == COMPLETED`): outcome status badge, summary paragraph, stats (total actions / blocked / confirmed).
- Status pill colours: SUBMITTED=muted, RUNNING=yellow, AWAITING_CONFIRMATION=orange, COMPLETED=green, HALTED=muted, FAILED=red.
- Outcome badge colours: SUCCESS=green, PARTIAL=yellow, FAILED=red, HALTED=muted.
- Action type chip colours: SCREENSHOT=muted, CLICK=blue, TYPE=blue, SCROLL=blue, KEY_PRESS=blue, NAVIGATE=purple, FILE_READ=teal, FILE_WRITE=orange, FILE_DELETE=red, FORM_SUBMIT=orange, ACCOUNT_MODIFY=red, API_CALL=orange.
