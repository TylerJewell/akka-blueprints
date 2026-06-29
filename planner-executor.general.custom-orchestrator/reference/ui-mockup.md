# UI mockup — custom-orchestration-agent

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Custom Orchestration Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` alone is not sufficient (Lesson 26).

## Mermaid CSS overrides (Lesson 24)

The `<style>` block includes the CSS overrides AND `themeVariables` from Lesson 24 — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these, the state-machine diagram renders state names invisible and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Custom Orchestration <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a task in the App UI tab, optionally swap the active strategy, watch the orchestrator route through tools.
- Card **How it works**: one paragraph naming the components, the routing loop, and the strategy-swap mechanism.
- Card **Components**: table with one row per component from `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/strategies/*` and `/api/control/*` routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (2 agents, 1 workflow, 4 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text and chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `hotl` muted (`HO1`), `sanitizer` green (`S1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">The strategy routes it.</span>` Subtitle: `The simulator drips a task every 90 s so the page is never empty.`
- **Strategy selector card** (top of the App UI tab): a dropdown populated from `GET /api/strategies`, showing each strategy name with a `(active)` badge on the current selection. A `Set active` button calls `POST /api/strategies/activate/{name}`. Reflects `strategy-update` SSE events live.
- **Task form card**: text field labelled "Task prompt", `Submit` button (yellow).
- **Operator controls pane** (top right): shows current halt state.
  - If `halted=false`: yellow `Halt new dispatches` button with a free-text reason field.
  - If `halted=true`: muted `HALTED` pill with reason and timestamp, plus a `Resume` button.
  - Reflects every `control-update` SSE event live.
- **Live task list**: cards per task; left border coloured by status (INITIALIZING = muted, EXECUTING = blue, COMPLETED = green, FAILED = red, HALTED = orange, STUCK = pale red).
  - Header row: prompt (first 80 chars), status pill, route iteration count, elapsed time.
  - Click to expand:
    - **Active strategy**: strategy name pill + maxRouteIterations / maxRevisits / allowedTools.
    - **Execution trace**: vertical timeline of `TraceEntry` rows. Each row shows sequence number, kind pill (`TOOL_CALL` blue, `REVISIT` purple, `BLOCKED` yellow), tool name (if applicable), verdict pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `FAILED` red), and a collapsed `<pre>` of `scrubbedResult` (click to expand). Redacted spans render in italics with a tooltip showing the redaction tag (e.g., `[REDACTED:aws-access-key]`).
    - **Answer** (only when status is COMPLETED): summary paragraph + citations list.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
