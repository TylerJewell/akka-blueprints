# UI mockup — durable-agent-baseline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Durable Agent Baseline</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Durable Agent <span class="accent">Baseline</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded work-order template (Data Pipeline / Code Analysis / Report Assembly) and click **Load example**.
  3. Click **Submit**.
  4. Watch the card progress through RUNNING → STEP_COMPLETED (×N) → COMPLETED → EVALUATED.
- Card **How it works**: one paragraph on submit → workflow initiates → agent runs all steps → performance eval fires; one paragraph on the two governance mechanisms (runtime monitor, periodic evaluator).
- Card **Components**: rows per component (WorkOrderEntity, RuntimeMonitor, WorkOrderWorkflow, WorkOrderAgent, PerformanceEvaluator, WorkOrderView, WorkOrderEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Scorer as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = autonomous` and `oversight.human_on_loop = true` declarations are the distinctive answers. `pii: false` is filled and prominent. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (M1, E1). ID badges coloured: M1 orange (hotl monitor), E1 blue (eval-periodic).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a work order. <span class="accent">Watch it run.</span>`. Subtitle: `One agent, durable across restarts.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Work-order template` (Data Pipeline / Code Analysis / Report Assembly / custom), `Title` text input, steps list editor (add/remove step rows, each with `stepId`, `description`, `maxRetries`), `Submitted by` text input, and a yellow `Submit` button. A **Load example** link fills all fields from the selected template.
    - Live list below: one card per work order, newest-first. Each card shows status pill, resolution badge (when result lands), eval score chip (when eval lands), title, step count, age.
  - **Right column** — Selected work-order detail.
    - Header: status pill + resolution badge + eval score chip + `title`.
    - Steps progress: a numbered list where each row shows the `stepId`, status icon (pending / running spinner / green check / red X), `retryCount` badge, `output` text, and elapsed time.
    - Alert section (shown only when alerts exist): one badge per alert with `kind` (STALL / RESOURCE_OVER_RUN) and `detail` text.
    - Result summary: the agent's 1–3-sentence paragraph and resolution enum.
    - Eval section at bottom: a 1–5 score chip and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: INITIATED=muted, RUNNING=yellow, STEP_COMPLETED=blue, COMPLETED=green, EVALUATED=green, STALLED=orange, FAILED=red.
- Resolution badge colours: COMPLETED=green, PARTIAL=yellow, FAILED=red.
- Alert badge colours: STALL=orange, RESOURCE_OVER_RUN=red.

The right pane's steps list updates incrementally via SSE — each `STEP_COMPLETED` event updates only the relevant row, so the user sees steps completing one by one rather than the whole list refreshing.
