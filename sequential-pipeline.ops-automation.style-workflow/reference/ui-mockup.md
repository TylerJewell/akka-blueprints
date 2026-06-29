# UI mockup — workflow-orchestration

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Workflows Process Orchestration</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Workflows Process <span class="accent">Orchestration</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded workflow definitions (or type a workflow id) and click **Run workflow**.
  3. Watch the card transition through VALIDATING → VALIDATED → EXECUTING → EXECUTED → NOTIFYING → NOTIFIED → EVALUATED.
  4. Inspect the rejection-log strip on the card if any stage-gate rejections fired.
- Card **How it works**: one paragraph on the three task stages (VALIDATE → EXECUTE → NOTIFY) and the typed handoff between them; one paragraph on the two governance mechanisms (stage-gate guardrail, step-coverage eval).
- Card **Components**: rows per component (WorkflowRunEntity, WorkflowRunPipeline, PipelineOrchestrationAgent, ValidateTools, ExecuteTools, NotifyTools, StageGuardrail, StepCoverageScorer, WorkflowRunView, WorkflowRunEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the workflow definition id is the only user input; step simulation comes from an in-process catalog). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, E1). ID badges coloured: H1 red (guardrail / hitl), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a workflow. <span class="accent">Review the run.</span>`. Subtitle: `One agent, three task stages, one runtime gate between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Workflow ID` (with a "Pick a seeded workflow" dropdown that fills it), and a yellow `Run workflow` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, eval score chip (when eval landed), workflow id, age, and a small red dot if any guardrail rejection fired during this run.
  - **Right column** — Selected-run detail.
    - Header: status pill + eval score chip + workflow id.
    - Stage panel 1 (Validation report): a table with columns stepId, status, warning. Visible once `validation.isPresent()`.
    - Stage panel 2 (Execution outcomes): a step-outcomes list with stepId, status pill, durationMs, detail. Visible once `execution.isPresent()`.
    - Stage panel 3 (Run result): summary counters (total / succeeded / failed / skipped), notification receipt (channel + messageId + sentAt). Visible once `result.isPresent()`.
    - Eval section at bottom: a 1–5 score widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the run has any `guardrailRejections`): a small table with stage, tool, reason, time.
- Status pill colours: CREATED=muted, VALIDATING=blue, VALIDATED=blue, EXECUTING=yellow, EXECUTED=yellow, NOTIFYING=blue, NOTIFIED=blue, EVALUATED=green, FAILED=red.

Each stage panel renders only when its data is present on the row record. A run in `EXECUTING` shows panels 1 and 2 (panel 2 with a "in progress" spinner if the agent has not yet returned). This is the visual proof that the typed handoff between stages is the only path information travels.
