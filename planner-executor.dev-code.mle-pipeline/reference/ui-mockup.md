# UI mockup — mle-pipeline

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from the blueprint authoring guide.

Browser title: `<title>Akka Sample: ML Engineering Pipeline</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` from Lesson 24 — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state-machine diagram on the Architecture tab renders state names invisible and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `ML Engineering <span class="accent">Pipeline</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a pipeline run in the App UI tab, watch the planner stage and dispatch, expand the row to see both ledgers and the final model report.
- Card **How it works**: one paragraph naming the components, the loop steps, and the four specialist agents.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/control/*` operator routes and the `/api/alerts` drift-alert route.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (5 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 3 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text, driver sublabel, and chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `ci-gate` blue (`G1`), `eval-periodic` amber (`E1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a run. <span class="accent">Watch the pipeline execute.</span>` Subtitle: `The simulator drips a run every 90 s so the page is never empty.`
- Form card: dataset reference field, objective text field, `Submit` button (yellow).
- **Operator controls pane** (top right of App UI tab): a card showing current halt state.
  - If `halted=false`: yellow `Halt new dispatches` button, a free-text reason field.
  - If `halted=true`: muted `HALTED` pill with reason and timestamp, plus a `Resume` button.
  - The pane reflects every `control-update` SSE event live.
- **Drift alerts banner**: below the operator pane, a collapsible list of recent `DriftAlert` entries across all runs. Each entry shows the run id (truncated), metric name, baseline, observed, delta, and detection time. Entries with delta > 0.15 get a red border; delta 0.05–0.15 get amber. Updated live via `drift-alert` SSE events.
- Live list: cards per run; left border coloured by status (PLANNING = muted, EXECUTING = blue, COMPLETED = green, GATE_FAILED = amber, FAILED = red, HALTED = orange, STUCK = pale red).
  - Header row: objective (first 80 chars), dataset ref, status pill, stage count, elapsed time. Drift warning badge when `driftAlerts` is non-empty.
  - Click to expand:
    - **Pipeline ledger**: facts (yellow list), gaps (muted list), stages (numbered list), activeDispatch (specialist pill + stage name + instruction line).
    - **Evaluation ledger**: vertical timeline of `EvalEntry` rows. Each row shows attempt number, specialist pill (`DATA_PROFILER` teal, `FEATURE_ENGINEER` purple, `MODEL_TRAINER` blue, `MODEL_EVALUATOR` green), stage name, verdict pill (`OK` green, `GATE_FAILED` amber, `FAILED` red, `BLOCKED` yellow), gate outcome badge, and a collapsed metric summary (accuracy / F1 / AUC / fairnessDelta) when `metrics` is present.
    - **Drift alerts** (when non-empty): list of `DriftAlert` rows with metric name, baseline → observed, and delta magnitude bar.
    - **Model report** (only when status is COMPLETED): summary paragraph + final metrics table (accuracy, F1, AUC, fairnessDelta) + stage list.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
