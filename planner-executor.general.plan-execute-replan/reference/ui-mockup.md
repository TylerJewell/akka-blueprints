# UI mockup — plan-execute-replan

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from the blueprint authoring guide.

Browser title: `<title>Akka Sample: Plan-and-Execute Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See Lesson 26 for the failure mode this prevents.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state-machine diagram on the Architecture tab renders state names invisible (black-on-black) and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Plan-and-Execute <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a goal in the App UI tab, watch the planner produce a plan, follow the executor and replanner iterate.
- Card **How it works**: one paragraph naming the three agents, the loop steps, and the two governance wires (guardrail and quality evaluator).
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (3 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
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
- ID badges coloured per mechanism: `guardrail` red (`G1`), `eval-event` blue (`E1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a goal. <span class="accent">Watch the loop run.</span>` Subtitle: `The simulator drips a goal every 90 s so the page is never empty.`
- Form card: text field labelled "Goal", `Submit` button (yellow).
- **Operator controls pane** (top right of the App UI tab): a card showing the current pause state.
  - If `paused=false`: yellow `Pause execution` button, a free-text reason field.
  - If `paused=true`: muted `PAUSED` pill with the reason and timestamp, plus a `Resume` button.
  - The pane reflects every `control-update` SSE event live.
- Live list: cards per goal; left border coloured by status (PLANNING = muted, EXECUTING = blue, CONCLUDED = green, FAILED = red, PAUSED = orange, STUCK = pale red).
  - Header row: goal text (first 80 chars), status pill, revision count, elapsed time.
  - Click to expand:
    - **Plan**: numbered list of `PlanStep` rows; each shows stepIndex, description, tool kind pill (`SEARCH` blue, `READ` green, `CALCULATE` purple, `SUMMARISE` muted), and the argument.
    - **Observations**: vertical timeline of `Observation` rows. Each row shows stepIndex, type pill (`STEP_OK` green, `STEP_FAILED` red, `STEP_BLOCKED` yellow, `PLAN_REVISED` orange, `EVAL` muted), a collapsed `<pre>` of `content` (click to expand), and — for EVAL rows — the score badge (≥ 70 green, 40–69 yellow, < 40 red) plus the rationale sentence.
    - **Conclusion** (only when status is CONCLUDED): summary paragraph + citation bullets.
    - **Failure or pause reason** (when applicable): one-paragraph block coloured by status.
