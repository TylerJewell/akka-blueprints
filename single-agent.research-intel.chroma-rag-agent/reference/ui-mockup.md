# UI mockup — chroma-rag-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: ChromaRagAgent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Chroma<span class="accent">RagAgent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded question (entity lifecycle / workflow timeout / view limitations) or type your own.
  3. Click **Ask**.
  4. Watch the card transition through RETRIEVING → ANSWERING → ANSWERED → EVALUATED.
- Card **How it works**: one paragraph on submit → retrieve → answer → eval; one paragraph on the governance mechanism (citation-groundedness guardrail).
- Card **Components**: rows per component (QueryEntity, CorpusIndexer, QueryWorkflow, RagAgent, CitationGuardrail, GroundednessScorer, RagView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer + 1 ChromaDbClient as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = false` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the grounded answer.</span>`. Subtitle: `One agent, one citation guardrail.`
- Layout: two-column.
  - **Left column** — Question panel + live list.
    - Question panel: dropdown `Seeded questions` (Entity lifecycle / Workflow timeouts / View query limits / custom), `Question` textarea (pre-filled from dropdown, editable), `Submitted by` text input, and a yellow `Ask` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, groundedness score chip (when eval landed), first 80 characters of question, age.
  - **Right column** — Selected-query detail.
    - Header: status pill + groundedness score chip + truncated question.
    - Retrieved chunks: a collapsible accordion with one row per chunk (chunkId, documentTitle, relevance score bar, passage excerpt).
    - Answer text: the agent's 1–4-sentence response paragraph.
    - Citations table: columns chunkId, documentTitle, passage (italic).
    - Groundedness section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, RETRIEVING=blue, ANSWERING=yellow, ANSWERED=blue, EVALUATED=green, FAILED=red.
- Groundedness score chip: score 4–5=green, score 3=yellow, score 1–2=red.

The raw chunks accordion is collapsed by default; the user expands it to inspect what passages the agent was given. This makes the retrieval step visible without cluttering the default view.
