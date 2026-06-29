# UI mockup — multi-step-query-engine

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Multi-Step Query Engine</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Multi-Step <span class="accent">Query Engine</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded questions (or type your own) and click **Run query**.
  3. Watch the card transition through DECOMPOSING → DECOMPOSED → RETRIEVING → RETRIEVED → SYNTHESIZING → SYNTHESIZED → EVALUATED.
  4. Inspect the eval score chip and confidence badge on the card.
- Card **How it works**: one paragraph on the three task phases (DECOMPOSE → RETRIEVE → SYNTHESIZE) and the typed handoff between them; one paragraph on the on-decision evaluator (four checks, score 1–5, non-blocking).
- Card **Components**: rows per component (QueryEntity, QueryPipelineWorkflow, QueryAgent, DecomposeTools, RetrieveTools, SynthesizeTools, StoppingEvaluator, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes and 1 Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the question is the only user input; passages come from an in-process corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (E1). ID badge coloured blue (eval-event).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the answer.</span>`. Subtitle: `One agent, three task phases, evidence quality scored on every result.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: textarea `Research question` (with a "Pick a seeded question" dropdown that fills it), and a yellow `Run query` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, eval score chip (when eval landed), confidence badge (HIGH/MEDIUM/LOW/INSUFFICIENT, when synthesized), question preview, and age.
  - **Right column** — Selected-query detail.
    - Header: status pill + eval score chip + confidence badge + question.
    - Phase panel 1 (Sub-questions): a numbered list of sub-questions with their priority. Visible once `decomposition.isPresent()`.
    - Phase panel 2 (Retrieved passages): a table with columns source, sub-question, relevance, text snippet. Visible once `evidence.isPresent()`.
    - Phase panel 3 (Answer): summary paragraph, confidence badge, then per-section blocks (heading, body, cited passage chips). Visible once `answer.isPresent()`.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: CREATED=muted, DECOMPOSING=blue, DECOMPOSED=blue, RETRIEVING=yellow, RETRIEVED=yellow, SYNTHESIZING=blue, SYNTHESIZED=blue, EVALUATED=green, FAILED=red.
- Confidence badge colours: HIGH=green, MEDIUM=yellow, LOW=orange, INSUFFICIENT=red.

Each phase panel renders only when its data is present on the row record. A query in `RETRIEVING` shows panel 1 (sub-questions) and panel 2 with a spinner while the agent is mid-retrieval. This is the visual proof that the typed handoff between phases is the only path information travels.
