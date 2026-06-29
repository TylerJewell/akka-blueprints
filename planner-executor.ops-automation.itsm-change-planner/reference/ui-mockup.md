# UI mockup — itsm-change-planner

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: ITSM Change Management Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` from Lesson 24 — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these, state names are invisible (black-on-black) and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `ITSM Change Management <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a change request in the App UI tab, watch the CAB approval gate, then track each implementation step through the executor loop.
- Card **How it works**: one paragraph naming the four agents, the workflow steps (plan → CAB approval → guardrail-vetted execution → test → backout-on-failure), and the three governance controls.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the CAB approval/rejection routes and the `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (4 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state machine, ER) with Akka theme variables and the required CSS overrides.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and placeholder "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `hitl` muted (`H1`), `guardrail` red (`G1`), `halt` red (`HT1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a change request. <span class="accent">Track it through CAB and execution.</span>` Subtitle: `The simulator drips a change request every 90 s so the page is never empty.`
- Form card: text field labelled "Change summary", text field labelled "CI name", category selector (STANDARD / NORMAL / EMERGENCY), `Submit` button (yellow).
- **CAB approval pane** (shown when a change is in `AWAITING_CAB`): a card listing the pending change's summary and CI name, a `Reviewer name` text field, an optional `Comments` field, and two buttons — `Approve` (yellow) and `Reject` (muted red). Clicking either calls the appropriate endpoint for the selected change.
- **Operator controls pane**: shows the current halt state.
  - If `halted=false`: yellow `Halt execution` button, a free-text reason field.
  - If `halted=true`: muted `HALTED` pill with the reason and timestamp, plus a `Resume` button.
  - Reflects every `control-update` SSE event live.
- Live list: cards per change; left border coloured by status (PLANNING = muted, AWAITING_CAB = yellow, APPROVED = blue, EXECUTING = blue, ROLLING_BACK = orange, IMPLEMENTED = green, REJECTED = muted, ROLLED_BACK = orange, FAILED = red, HALTED = orange, STUCK = pale red).
  - Header row: summary (first 80 chars), status pill, CI name, category badge, elapsed time.
  - Click to expand:
    - **Impact assessment**: one-paragraph block.
    - **Implementation plan**: numbered list of steps with `targetCi` chip and `expectedOutcome` in muted text.
    - **Test plan**: numbered list matching implementation steps, each with `successCriteria`.
    - **Backout plan**: numbered list in reverse order with `targetCi` chip.
    - **Execution log**: vertical timeline of `StepRecord` rows. Each row shows sequence number, description, step result verdict pill (`OK` green / `FAILED` red), evidence in a collapsible `<pre>`, test result verdict pill (`PASSED` green / `FAILED` red), and observation text.
    - **CAB decision** (once set): outcome badge (APPROVED green / REJECTED red), reviewer name, comments, timestamp.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
