# UI mockup — durable-workflow-recovery

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Durable Workflows with DBOS</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Durable Workflow<span class="accent">Recovery</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded workflow type (ETL pipeline / Payment batch / Data migration) and load it with one click.
  3. Click **Register execution**.
  4. Watch the card transition through RUNNING → STALLED → ANALYZING → DECISION_RECORDED → HEALTH_SCORED.
- Card **How it works**: one paragraph on register → checkpoint → stall-detection → analysis → health-eval; one paragraph on the two governance mechanisms (graceful-degradation halt, periodic health evaluator).
- Card **Components**: rows per component (ExecutionEntity, CheckpointConsumer, RecoveryWorkflow, WorkflowRecoveryAgent, ResponseValidator, HealthEvaluator, ExecutionView, RecoveryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Validator + 1 Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = enforce` declaration and `oversight.human_in_loop = false` are the distinctive answers (recovery is automated; escalation is the human path). Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, E1). ID badges coloured: H1 orange (halt), E1 blue (eval-periodic).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Register an execution. <span class="accent">Read the recovery decision.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Registration panel + live list.
    - Registration panel: dropdown `Workflow type` (ETL pipeline / Payment batch / Data migration / custom), `Workflow ID` text input (auto-filled from type + date for seeded examples), `Owner team` text input, and a yellow `Register execution` button with a "Load seeded example" link below.
    - Live list below: one card per execution, newest-first. Each card shows status pill, verdict badge (when decision landed), health score chip (when health scored), workflow type label, age.
  - **Right column** — Selected-execution detail.
    - Header: status pill + verdict badge + health score chip + workflow ID.
    - Checkpoint timeline: a horizontal progress bar with one node per checkpoint (coloured by outcome: COMPLETED=green, FAILED=red, PENDING=yellow, SKIPPED=muted) plus elapsed-ms label below each node.
    - Snapshot summary: `totalExpected`, `completedCount`, `failedCount`, `stalledFor` duration chips.
    - Decision rationale: the agent's 2–4 sentence paragraph.
    - Checkpoint status table: columns checkpoint id, phase, outcome (coloured chip), recommended action.
    - Health section at bottom: a 1–5 score widget and the one-line diagnosis. Score ≤ 2 highlights the card border red.
- Status pill colours: REGISTERED=muted, RUNNING=blue, STALLED=yellow, ANALYZING=yellow, DECISION_RECORDED=blue, HEALTH_SCORED=green, ABORTED=red, FAILED=red.
- Verdict badge colours: RESUME=green, ESCALATE=yellow, ABORT=red.
- Checkpoint outcome chip colours: COMPLETED=green, PENDING=yellow, FAILED=red, SKIPPED=muted.
