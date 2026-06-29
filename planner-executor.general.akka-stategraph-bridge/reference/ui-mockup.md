# UI mockup — akka-stategraph-bridge

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Graph API Plugin</title>`.

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

The DOM must contain **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block includes CSS overrides AND `themeVariables` from Lesson 24 — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state machine diagram renders node labels invisible (black-on-black) and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Graph API <span class="accent">Plugin</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a graph definition in the App UI tab, watch the node execution loop, expand the row to see the plan, state, and trace.
- Card **How it works**: one paragraph naming the three agents, the workflow, the entity, and the execution loop (plan → execute-node → sanitize → record-state → route → check-cycle).
- Card **Components**: table with one row per component from `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: 3 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints.
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with question text, drives sublabel, and widgets in selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and placeholder "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `hotl` muted (`HO1`), `sanitizer` green (`S1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a graph. <span class="accent">Watch it execute.</span>` Subtitle: `The simulator drips a graph definition every 90 s so the page is never empty.`
- Form card: a JSON textarea labelled "Graph definition (JSON)", `Submit` button (yellow).
- **Operator controls pane** (top right of the App UI tab): a card showing current halt state.
  - If `halted=false`: yellow `Halt new dispatches` button + free-text reason field.
  - If `halted=true`: muted `HALTED` pill with reason and timestamp + `Resume` button.
  - Pane reflects every `control-update` SSE event live.
- Live list: cards per run; left border coloured by status (PLANNING = muted, EXECUTING = blue, COMPLETED = green, FAILED = red, HALTED = orange, STUCK = pale red).
  - Header row: first 80 chars of `graphJson`, status pill, node count, elapsed time.
  - Click to expand:
    - **Graph plan**: node list with id, description, tool annotation chip (if set), maxVisits badge. Edge list with from → to, predicate (if set), back-edge indicator. Entry and terminal node highlighted.
    - **Current state**: key/value grid of `GraphState.fields`.
    - **Execution trace**: vertical timeline of `TraceEntry` rows. Each row shows step index, node id chip, verdict pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `FAILED` red, `SANITIZED` blue), and a collapsed `<pre>` of `scrubbedContent` (click to expand). Redacted spans render in italics with tooltip showing the redaction tag.
    - **Result** (only when `COMPLETED`): summary paragraph + final state snapshot.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
