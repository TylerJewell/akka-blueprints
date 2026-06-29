# UI mockup — citation-rag

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Citation Query Engine</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Citation Query <span class="accent">Engine</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded queries (or type your own) and click **Run query**.
  3. Watch the card transition through RETRIEVING → RETRIEVED → ATTRIBUTING → ATTRIBUTED → COMPOSING → COMPOSED → EVALUATED.
  4. Inspect the citation score chip and the rejection-log strip on the card if any citation-gate rejections fired.
- Card **How it works**: one paragraph on the three task phases (RETRIEVE → ATTRIBUTE → COMPOSE) and the typed handoff between them; one paragraph on the two governance mechanisms (after-llm-response citation guardrail, citation coverage eval).
- Card **Components**: rows per component (QueryEntity, CitationQueryWorkflow, CitationAgent, RetrieveTools, AttributeTools, ComposeTools, CitationGuardrail, CoverageScorer, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the query is the only user input; passages come from an in-process corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Type a query. <span class="accent">Read the cited answer.</span>`. Subtitle: `One agent, three task phases, one citation gate at the output boundary.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Research query` (with a "Pick a seeded query" dropdown that fills it), and a yellow `Run query` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, citation score chip (when score landed), query text, age, and a small red dot if any guardrail rejection fired during this query.
  - **Right column** — Selected-query detail.
    - Header: status pill + citation score chip + query text.
    - Phase panel 1 (Retrieved passages): a table with columns source, document title, text preview, score. Visible once `passages` is present.
    - Phase panel 2 (Attributed claims): a claims list with claimId, text, cited passage reference, and confidence bar. Visible once `claimSet` is present.
    - Phase panel 3 (Composed answer): response body paragraph, then an inline citations table (claimId, passage source, snippet preview). Visible once `answer` is present.
    - Citation score section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the query has any `guardrailRejections`): a small table with claimId, reason, time.
- Status pill colours: CREATED=muted, RETRIEVING=blue, RETRIEVED=blue, ATTRIBUTING=yellow, ATTRIBUTED=yellow, COMPOSING=blue, COMPOSED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A query in `ATTRIBUTING` shows panels 1 and 2 (panel 2 with a "in progress" spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
