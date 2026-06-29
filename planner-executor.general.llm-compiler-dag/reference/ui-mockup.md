# UI mockup — llm-compiler-dag

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: LLMCompiler</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. Canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` from Lesson 24 — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these, state-machine diagram state names render invisible (black-on-black) and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `LLM<span class="accent">Compiler</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a query in the App UI tab, watch the Planner compile a DAG and the Task Fetching Unit dispatch in parallel, expand the row to see each tool call resolved and the final synthesized answer.
- Card **How it works**: one paragraph naming the Planner, the Task Fetching Unit's parallel frontier dispatch, the Joiner synthesis step, and the three governance controls.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (2 agents, 1 workflow, 3 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `hotl` muted (`HO1`), `sanitizer` green (`S1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a query. <span class="accent">Watch the DAG compile and dispatch.</span>` Subtitle: `The simulator drips a query every 90 s so the page is never empty.`
- Form card: text field labelled "Query", `Submit` button (yellow).
- **Operator controls pane** (top right of the App UI tab): a card showing the current halt state.
  - If `halted=false`: yellow `Halt new dispatches` button, free-text reason field.
  - If `halted=true`: muted `HALTED` pill with reason and timestamp, plus a `Resume` button.
  - The pane reflects every `control-update` SSE event live.
- Live list: cards per job; left border coloured by status (COMPILING = muted, RUNNING = blue, COMPLETED = green, FAILED = red, HALTED = orange, STALE = pale red).
  - Header row: query (first 80 chars), status pill, resolved/total tool call count, elapsed time.
  - Click to expand:
    - **Compilation plan**: a DAG card showing each `ToolCall` node as a chip (`callId` · tool-kind badge · status badge). Dependency arrows drawn between chips. Approved nodes show their tool kind in colour (`SEARCH` blue, `CALCULATOR` yellow, `LOOKUP` green, `CODE_EVAL` purple); `SKIPPED` nodes are muted with a strikethrough.
    - **Tool results**: vertical list of `ToolResult` rows. Each row shows `callId`, tool-kind badge, verdict pill (`OK` green, `BLOCKED` yellow, `FAILED` red), and a collapsed `<pre>` of `output` (click to expand). Redacted spans render in italics with a tooltip showing the redaction tag.
    - **Answer** (only when status is COMPLETED): summary paragraph + citation bullets, each tagged by `callId` and tool kind.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
