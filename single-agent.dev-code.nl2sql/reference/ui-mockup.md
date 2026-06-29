# UI mockup — nl2sql

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: NL2SQL</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `NL<span class="accent">2SQL</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded schema context (orders / customers / fulfillment) and a matching seed question, or type your own.
  3. Click **Run query**.
  4. Watch the card transition through SCHEMA_ATTACHED → TRANSLATING → RESULT_READY → SCORED.
- Card **How it works**: one paragraph on submit → schema-attach → translate-and-run → score; one paragraph on the two governance mechanisms (SQL safety guardrail, automatic-safety-halt).
- Card **Components**: rows per component (QueryEntity, SchemaRegistryConsumer, QueryWorkflow, SqlQueryAgent, SqlSafetyGuardrail, SqlSafetyHalt, QueryResultScorer, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Halt + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = automated-execution` and `oversight.human_in_loop = false` declarations are prominent (reads are low-risk when write prevention is enforced). Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 orange (halt).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Get the data.</span>`. Subtitle: `One agent, two safety layers around every query.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Schema context` (orders / customers / fulfillment / full schema), `Question` textarea (with a "Load seed question" link that fills the textarea), `Submitted by` text input, and a yellow `Run query` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, score chip (when scored), question text (truncated to 60 chars), age.
  - **Right column** — Selected-query detail.
    - Header: status pill + score chip + question text.
    - Schema fragment preview: a collapsible list of tables with column names and types.
    - Generated SQL: a monospace code block with syntax-highlight-style colouring (keywords bold/blue).
    - Result table: paginated data table with column headers and typed cell values. Row count badge above the table.
    - Score section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border yellow.
    - Halt banner: if status is HALTED, a red banner shows the `haltReason` text.
- Status pill colours: SUBMITTED=muted, SCHEMA_ATTACHED=blue, TRANSLATING=yellow, RESULT_READY=blue, SCORED=green, HALTED=red, FAILED=red.
- Score chip colours: 4–5=green, 3=blue, 1–2=yellow.
- The generated SQL is always shown regardless of status so the user can inspect what was attempted (even in HALTED state the partial SQL that triggered the halt is visible if available).
