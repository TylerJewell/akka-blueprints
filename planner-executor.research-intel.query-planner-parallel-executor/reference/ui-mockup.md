# UI mockup — query-planner-parallel-executor

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from Section 13 of the authoring guide.

Browser title: `<title>Akka Sample: Query Planning Workflow</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler switches tabs by matching these attributes — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements. Any panel removed in an earlier iteration must be deleted from the HTML — `display:none` is not enough. See Lesson 26.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes CSS overrides and `themeVariables` from Lesson 24 — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state machine on the Architecture tab renders state names invisible and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Query Planning <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a research question in the App UI tab, watch the planner decompose and executors run in parallel, expand the row to see the plan, per-round results, and the final research answer.
- Card **How it works**: one paragraph naming the components, the plan → eval → fan-out → coverage loop, and the three retrieval executor types.
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
- Each `.qb` rendered with the question text and answer widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `eval-event` teal (`E1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a research question. <span class="accent">Watch the plan fan out.</span>` Subtitle: `The simulator drips a question every 90 s so the page is never empty.`
- Form card: text area labelled "Research question", `Submit` button (yellow).
- **Operator controls pane** (top right of the App UI tab): shows current halt state.
  - If `halted=false`: yellow `Halt new dispatches` button with a reason field.
  - If `halted=true`: muted `HALTED` pill with reason and timestamp, plus a `Resume` button.
  - Pane reflects every `control-update` SSE event live.
- Live list: cards per session; left border coloured by status (PLANNING = muted, EVALUATING = teal, EXECUTING = blue, SYNTHESIZING = purple, COMPLETED = green, FAILED = red, HALTED = orange, STALE = pale red).
  - Header row: question (first 80 chars), status pill, round number, elapsed time.
  - Click to expand:
    - **Current plan**: `coverageGoal` line, then a numbered list of sub-queries — each shows `subQueryId`, strategy pill (`CORPUS` yellow, `WEB` blue, `KNOWLEDGE_BASE` green), and `queryText`.
    - **Plan evaluation**: `PlanEvaluation` card showing score bar (0–1), `passing` badge, and `rationale`.
    - **Results** (grouped by round): a table per round. Each row shows `subQueryId`, strategy pill, `queryText` (first 60 chars), `verdict` pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `FAILED` red, `TIMED_OUT` muted), and a collapsed `<pre>` of `content` (click to expand). Redacted spans render in italics with a tooltip.
    - **Research answer** (only when `status = COMPLETED`): summary paragraph + confidence bar + citation bullets. Confidence badge: green ≥ 0.8, yellow ≥ 0.5, red < 0.5.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
