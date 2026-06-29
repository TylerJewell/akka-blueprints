# UI mockup — field-service-dispatcher

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Scheduling Operations Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index.

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient (Lesson 26).

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state-machine diagram on the Architecture tab renders state names invisible (black-on-black) and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Scheduling Operations <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a work order in the App UI tab, watch the dispatcher iterate, expand the row to see the schedule ledger, fairness ledger entries, and the dispatch summary.
- Card **How it works**: one paragraph naming the components, the planner-executor loop steps, and the two specialists.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (3 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 3 timed-actions, 2 endpoints).
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
- ID badges coloured per mechanism: `guardrail` red (`G1`), `eval-periodic` blue (`E1`), `hotl` muted (`HO1`), `sanitizer` green (`S1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a work order. <span class="accent">Watch dispatch run.</span>` Subtitle: `The simulator drips a work order every 90 s so the page is never empty.`
- Form card: fields for "Work order description" (text), "Service address" (text), "Required skill" (text). `Submit` button (yellow).
- **Operator controls pane** (top right of the App UI tab): a card showing the current pause state.
  - If `paused=false`: yellow `Pause new dispatches` button, a free-text reason field.
  - If `paused=true`: muted `PAUSED` pill with the reason and timestamp, plus a `Resume` button.
  - The pane reflects every `control-update` SSE event live.
- **Fairness alerts pane** (below operator controls): shows any active `FairnessAlert` records from the most recent schedules. Each alert shows technician id, alert type, detail text, and detected-at timestamp. Alerts clear when no new alert has fired in the last 5 minutes.
- Live list: cards per schedule; left border coloured by status (PLANNING = muted, DISPATCHING = blue, COMPLETED = green, FAILED = red, PAUSED = orange, STALLED = pale red).
  - Header row: description (first 80 chars), status pill, assignment count, elapsed time.
  - Click to expand:
    - **Schedule ledger**: knownOrders (yellow list), unassignedSlots (muted list), plan (numbered list), currentDispatch (specialist pill + assignment line).
    - **Fairness ledger**: vertical timeline of `AssignmentEntry` rows. Each row shows attempt number, specialist pill (`ROUTE_OPTIMIZER` purple, `AVAILABILITY` blue), assignment text, verdict pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `FAILED` red, `FAIRNESS_FLAGGED` orange), and a collapsed `<pre>` of `scrubbedResult` (click to expand). Redacted spans render in italics with a tooltip showing the redaction tag.
    - **Fairness alerts** (inline, if any): each `FairnessAlert` record for this schedule, showing technician id, type, and detail.
    - **Dispatch summary** (only when status is COMPLETED): overview paragraph + assignments bullets.
    - **Failure or pause reason** (when applicable): one-paragraph block coloured by status.
