# UI mockup — activity-interrupt-cancellation

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Activity Interrupt / Cancellation</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements: Overview, Architecture, Risk Survey, Eval Matrix, App UI. Any panel removed during iteration must be deleted from the HTML; `display:none` is not sufficient. See AKKA-EXEMPLAR-LESSONS.md Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Activity Interrupt / <span class="accent">Cancellation</span>`. No subtitle.
- Card **Try it**: `/akka:build` block, then 4 numbered steps (open App UI, wait for the simulator to drop a task, watch step progress, click Cancel mid-run).
- Card **How it works**: one paragraph on the poll → queue → workflow → agent-loop → cancel-signal flow; one paragraph on the two halt mechanisms (operator stop + graceful degradation).
- Card **Components**: rows per component (poller, queue, dispatcher, runner agent, cleanup agent, workflow, entity, view, reaper, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Agent, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs with answers from `risk-survey.yaml`. The `human_on_loop: true` and `reviewer_can_cancel_any_running_task: true` declarations are the most distinctive. Most data_classes fields are `false` (no PII); `infrastructure-credentials` is `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, H2). ID badges coloured: H1 and H2 both pink/red (halt mechanism).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Run tasks. <span class="accent">Cancel any time.</span>`. Subtitle: `Simulated tasks drop every 20 s. Cancel mid-flight to see the cleanup path.`
- Layout: two-column.
  - **Left column** — Live activity list, sorted newest-first. Each card shows:
    - Status pill.
    - Kind badge (INFRA, REPORT, PIPELINE, MAINT).
    - Task name.
    - Step progress indicator (e.g., "Step 2 of 5").
    - Cancel button — active when RUNNING or QUEUED; greyed when terminal.
  - **Right column** — selected task detail:
    - Task description (read-only).
    - Step log timeline: each step as a row with index, summary, artefact chip (when present), and timestamp.
    - Cancel button (red border). Clicking opens a small "Reason" input and a Confirm button.
    - Cleanup plan block — visible only after CANCELLED. Lists the plan's ordered steps and the reason.
    - Stale-timeout badge when `cancellationRequest.requestedBy == "reaper"`.
- Status pill colours: QUEUED=muted, RUNNING=blue, CANCELLING=orange, CANCELLED=red, COMPLETED=green, FAILED=dark-red.
