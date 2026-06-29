# UI mockup — sequential-workflow

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Sequential Workflow</title>`.

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

And the DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Sequential <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded job types (or fill in your own) and click **Run workflow**.
  3. Watch the card transition through VALIDATING → VALIDATED → ENRICHING → ENRICHED → EXECUTING → EXECUTED → SUMMARIZING → SUMMARIZED → EVALUATED.
  4. Inspect the rejection-log strip on the card if any phase-gate rejections fired.
- Card **How it works**: one paragraph on the four task phases (VALIDATE → ENRICH → EXECUTE → SUMMARIZE) and the typed handoff between them; one paragraph on the two governance mechanisms (phase-gate guardrail, completeness eval).
- Card **Components**: rows per component (JobEntity, JobPipelineWorkflow, WorkflowAgent, ValidateTools, EnrichTools, ExecuteTools, SummarizeTools, StepGuardrail, QualityScorer, JobView, JobEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 4 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the job definition is the only user input; job-type catalog data comes from an in-process resource). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a job. <span class="accent">Review the result.</span>`. Subtitle: `One agent, four task phases, one runtime gate between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Job name`, dropdown `Job type` (with seeded options: `csv-transform`, `report-generation`, `data-validation-run`), and a yellow `Run workflow` button.
    - Live list below: one card per job, newest-first. Each card shows status pill, quality score chip (when eval landed), job name, age, and a small red dot if any guardrail rejection fired during this job.
  - **Right column** — Selected-job detail.
    - Header: status pill + quality score chip + job name.
    - Phase panel 1 (Validation): a table with columns fieldName, status, note. Visible once `validation.isPresent()`.
    - Phase panel 2 (Enrichment): a parameters list with key/value/source and a metadata map. Visible once `enrichedJob.isPresent()`.
    - Phase panel 3 (Execution): a step list with stepId, stepName, outcome, and expandable artifact chips per step. Visible once `output.isPresent()`.
    - Phase panel 4 (Summary): title, outcome statement, per-entry block (heading, body, artifact id chips). Visible once `summary.isPresent()`.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the job has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: CREATED=muted, VALIDATING=blue, VALIDATED=blue, ENRICHING=yellow, ENRICHED=yellow, EXECUTING=orange, EXECUTED=orange, SUMMARIZING=blue, SUMMARIZED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A job in `EXECUTING` shows panels 1 and 2 (panel 3 with a "in progress" spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
