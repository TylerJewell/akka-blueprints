# UI mockup — self-discover-planner

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from Section 13 of the blueprint-authoring guide.

Browser title: `<title>Akka Sample: Self-Discover Workflow</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed during iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state-machine diagram renders state names invisible (black-on-black) and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Self-Discover <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a task in the App UI tab, watch the planner compose a plan and the evaluator score it, expand the row to see the accepted plan, per-step execution log, and the synthesised answer.
- Card **How it works**: one paragraph naming the four agents, the plan-evaluation gate, the module execution loop, and the scrubber.
- Card **Components**: table with one row per component from `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including `/api/registry`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (4 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state machine, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with question text, drives sublabel, and chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badge for `E1` coloured blue (eval-event category).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Watch the plan emerge.</span>` Subtitle: `The simulator drips a task every 90 s so the page is never empty.`
- Form card: text field labelled "Task prompt", `Submit` button (yellow).
- Live list: cards per solve; left border coloured by status (PLANNING = muted, EVALUATING = blue-grey, EXECUTING = blue, COMPLETED = green, FAILED = red, STUCK = pale red).
  - Header row: prompt (first 80 chars), status pill, step progress (`x / y modules`), elapsed time.
  - Click to expand:
    - **Reasoning plan** (shown once EVALUATING or beyond): `selectedModuleIds` as chips coloured by `ModuleKind` (DECOMPOSE = yellow, ANALYSE = blue, COMPARE = teal, GENERATE = purple, VERIFY = green, REFLECT = muted), step list with `objective` and `inputsFrom` arrows, `rationale` paragraph, `planRevisionCount` badge.
    - **Plan eval** (shown once available): `score` as a bar (green ≥ 0.65, red < 0.65), `verdict` pill (PASS green, FAIL red), `feedback` sentence. If a prior plan was rejected, a "revision N" accordion shows the earlier eval.
    - **Execution log** (shown from EXECUTING onwards): vertical timeline of `ExecutionStep` rows. Each row shows `stepIndex`, `ModuleKind` pill, `objective`, a collapsed `<pre>` of `result.output` (click to expand). Redacted spans render in italics with a tooltip showing the redaction tag.
    - **Answer** (only when status is COMPLETED): `summary` paragraph + `evidence` bullets.
    - **Failure reason** (when status is FAILED or STUCK): one-paragraph block coloured by status.
