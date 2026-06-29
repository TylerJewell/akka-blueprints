# UI mockup — project-tracker

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Project Manager Agent (Planner)</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the blank-tab failure mode this avoids.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Project Manager <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the simulator to drop a task, watch it get assigned, observe a nudge after the due date passes).
- Card **How it works**: one paragraph on the poll → normalise → assign → guardrail → dispatch flow; one paragraph on the before-tool-call guardrail and how it gates every action that touches a human.
- Card **Components**: rows per component (poller, queue, consumer, assignment agent, nudge agent, eval judge, workflow, guardrail checker, entity, view, stale checker, eval runner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 3 TimedActions, 2 HttpEndpoints, 1 GuardrailChecker).
- Four mermaid cards (component graph, sequence, state machine, entity model).
- Compressed comp-row table.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs covering Purpose, Data, Decisions, Failure, Oversight, Operations, Compliance — answers from `risk-survey.yaml`. The `employee-performance-data: true` declaration in Data is a notable filled chip. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured yellow (guardrail).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the board. <span class="accent">Guardrail gates every action.</span>`. Subtitle: `Simulated tasks arrive every 20 s. Overdue tasks get nudged automatically.`
- Layout: two-column.
  - **Left column** — Live task list, sorted OVERDUE/NUDGED/ESCALATED first, then CREATED/ASSIGNED/IN_PROGRESS, then COMPLETED/CANCELLED. Each card shows:
    - Status pill.
    - Priority badge (URGENT=red, HIGH=orange, NORMAL=muted, LOW=dim).
    - Task title and assignee chip (if assigned).
    - Due date (highlighted in red if past).
    - Nudge count badge (shown when nudgeCount > 0).
  - **Right column** — selected task detail.
    - Full task description (read-only).
    - Assignment recommendation card: owner name, rationale, confidence chip.
    - Nudge history list (each entry: tone chip, message preview, timestamp).
    - Complete button (green). Escalate button (border, red text). Escalate opens a small "Reason" textarea.
    - After completion: eval score chip (1–5) and rationale text.
- Status pill colours: CREATED=muted, ASSIGNED=blue, IN_PROGRESS=blue, OVERDUE=orange, NUDGED=yellow, ESCALATED=red, COMPLETED=green, CANCELLED=muted.

Guardrail block events are surfaced in the detail pane as an amber "Assignment blocked" banner with the `blockReason` text.
