# UI mockup — bq-pipeline-builder

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from the blueprint authoring guide.

Browser title: `<title>Akka Sample: BigQuery Pipeline Builder</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state-machine diagram on the Architecture tab renders state names invisible (black-on-black) and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `BigQuery <span class="accent">Pipeline Builder</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — describe a pipeline in the App UI tab, watch the planner iterate through build steps, expand the row to see both ledgers and the final build manifest.
- Card **How it works**: one paragraph naming the components, the loop steps, and the four specialist executors.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (5 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text, drives sublabel, and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `ci-gate` blue (`CI1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Describe a pipeline. <span class="accent">Watch it get built.</span>` Subtitle: `The simulator drips a pipeline request every 90 s so the page is never empty.`
- Form card: text field labelled "Pipeline description", `Submit` button (yellow).
- **Operator controls pane** (top right of the App UI tab): a card showing the current halt state.
  - If `halted=false`: yellow `Halt new dispatches` button, a free-text reason field.
  - If `halted=true`: muted `HALTED` pill with the reason and timestamp, plus a `Resume` button.
  - The pane reflects every `control-update` SSE event live.
- Live list: cards per pipeline; left border coloured by status (PLANNING = muted, BUILDING = blue, COMPLETED = green, FAILED = red, HALTED = orange, STUCK = pale red).
  - Header row: description (first 80 chars), status pill, step count, elapsed time.
  - Click to expand:
    - **Pipeline ledger**: dataset facts (yellow list), schema gaps (muted list), build plan (numbered list), currentDispatch (executor pill + step line).
    - **Step ledger**: vertical timeline of `StepEntry` rows. Each row shows attempt number, executor pill (`SCHEMA_ANALYST` teal, `SQL_COMPOSER` blue, `DATAFORM_MODELER` purple, `VALIDATOR` muted), step text, verdict pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `FAILED` red, `CI_GATE_FAILED` orange), and a collapsed `<pre>` of `output` (click to expand).
    - **Build manifest** (only when status is COMPLETED): summary paragraph + artifact bullets.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
