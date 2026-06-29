# UI mockup — parallel-voting-workflow

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Parallelization Workflow</title>`.

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

## Mermaid theming — MUST carry the Lesson 24 overrides

The Architecture tab's `stateDiagram-v2` needs explicit CSS so state names render white-on-dark and edge labels are not clipped. The `<style>` block must set `color`/`fill` to `#ffffff` on every state-label DOM path and `overflow:visible` on every edge-label `foreignObject`, and `mermaid.initialize` must set `nodeTextColor`, `stateLabelColor`, and `transitionLabelColor: #cccccc`. Without both, state names render black-on-black and transition labels clip at the top.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Parallelization <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: just the `/akka:build` (Claude Code) block — no env-var export block — then three numbered steps (submit a task, watch it progress to DECIDED, expand it for the three voter ballots + aggregated decision + alignment score).
- Card **How it works**: one paragraph naming the components and the parallel three-voter fan-out aggregated by the dispatcher.
- Card **Components**: table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (5 agents, 1 workflow, 2 entities, 1 view, 1 consumer, 2 timed actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables and the Lesson 24 overrides.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text, drives sublabel, and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer"); unanswered `.qb` blocks get `opacity: 0.45`.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badge coloured per mechanism: eval-event = blue.
- Rows expand vertically on click; one open at a time, showing rationale + implementation from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Watch three voters decide.</span>` Subtitle: `The simulator also drips a task every 60 s so the page is never empty.`
- Form card: a text field labelled "Title", a textarea labelled "Task description", a `Submit` button (yellow).
- Live list: cards per task; left border coloured by status (SUBMITTED = muted, VOTING = blue, DECIDED = green, PARTIAL = orange, INCONCLUSIVE = red).
  - Header row: title, status pill, and an optional alignment-score chip.
  - Click to expand: three voter blocks (Feasibility / Risk / Alignment) each showing the ballot, confidence, and reasons list with weight badges; then the aggregated decision block (decision + summary + quorumMet); then the alignment rationale if scored.
