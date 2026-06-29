# UI mockup — json-query-engine

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: JSON Query Engine Workflow</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The exemplar's canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `JSON Query Engine <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded document from the dropdown, enter a question (or use one of the suggested questions), and click **Run query**.
  3. Watch the card transition through PARSING → PARSED → TRAVERSING → TRAVERSED → RESPONDING → RESPONDED → EVALUATED.
  4. Inspect the path-rejection log strip on the card if any path-validation rejections fired.
- Card **How it works**: one paragraph on the three task phases (PARSE → TRAVERSE → RESPOND) and the typed handoff between them; one paragraph on the path-expression validation guardrail and the accuracy evaluator.
- Card **Components**: rows per component (QueryEntity, JsonQueryWorkflow, QueryAgent, ParseTools, TraverseTools, RespondTools, PathGuardrail, AccuracyScorer, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (question text and document IDs are the only user inputs; the JSON corpus is in-process configuration/product data). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the answer.</span>`. Subtitle: `One agent, three task phases, path expressions validated before every document read.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Question` with suggested-question links underneath (e.g., "What is the price of the Pro plan?", "Who leads the Engineering department?", "What port does the auth-service listen on?"), a document dropdown (`saas-pricing-catalog`, `team-directory`, `infrastructure-config`), and a yellow `Run query` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, accuracy score chip (when eval landed), question truncated to 60 chars, document id, age, and a small red dot if any path-validation rejection fired.
  - **Right column** — Selected-query detail.
    - Header: status pill + accuracy score chip + question text + document id badge.
    - Phase panel 1 (Parsed question): intent type badge, target entity, candidate root keys list. Visible once `parsed.isPresent()`.
    - Phase panel 2 (Traversal): a table with columns path expression, resolved value, depth level. Visible once `traversal.isPresent()`.
    - Phase panel 3 (Answer): the composed answer paragraph, then a citations section — one chip per citation showing path expression and value. Visible once `result.isPresent()`.
    - Accuracy section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Path-rejection log strip (only visible if the query has any `guardrailRejections`): a table with phase, tool, path expression, reason, time.
- Status pill colours: CREATED=muted, PARSING=blue, PARSED=blue, TRAVERSING=yellow, TRAVERSED=yellow, RESPONDING=blue, RESPONDED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A query in `TRAVERSING` shows panels 1 and 2 (panel 2 with a spinner while the agent is resolving paths). This is the visual proof that the typed handoff between phases is the only path information travels.
