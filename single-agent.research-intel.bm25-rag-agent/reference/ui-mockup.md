# UI mockup — bm25-rag-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: BM25 RAG Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `BM25 <span class="accent">RAG Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded question or type your own.
  3. Adjust the Top-K slider if desired, then click **Ask**.
  4. Watch the card transition through PASSAGES_ATTACHED → ANSWERING → ANSWER_RECORDED → SCORED.
- Card **How it works**: one paragraph on question → BM25 retrieval → agent answer → grounding score; one paragraph on the citation guardrail and grounding scorer.
- Card **Components**: rows per component (QueryEntity, BM25Retriever, CorpusIndex, QueryWorkflow, CorpusQueryAgent, CitationGuardrail, GroundingScorer, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer + 1 Index as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled and prominent. `decisions.authority_level = informational` and `oversight.human_in_loop = false` are the distinctive answers — this system surfaces references, not decisions. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- Row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the answer.</span>`. Subtitle: `One agent, one guardrail, BM25 retrieval grounding every citation.`
- Layout: two-column.
  - **Left column** — Question panel + live list.
    - Question panel: four seeded-question pills (click to fill), `Question` textarea, `Top-K` range input (1–10, default 5, shows current value), `Submitted by` text input, and a yellow `Ask` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, answer type badge (when answer landed), grounding score chip (when scored), question text (truncated), age.
  - **Right column** — Selected-query detail.
    - Header: status pill + answer type badge + grounding score chip + question text.
    - Retrieved passages: a compact table of passage ids, titles, BM25 scores, and snippets.
    - Answer section: answer type badge, the answer text, and a citation list (passage id chip → quoted fragment in italic).
    - Grounding section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, PASSAGES_ATTACHED=blue, ANSWERING=yellow, ANSWER_RECORDED=blue, SCORED=green, FAILED=red.
- Answer type badge colours: DIRECT=green, PARTIAL=yellow, NO_ANSWER=muted.
- Grounding score chip: 5=green, 4=blue, 3=yellow, 1–2=red.

The passage bodies are never displayed in full on this screen — only the snippet (first 200 chars). Developers who need the full body fetch `/api/queries/{id}` and read from the `passages` field. This keeps the UI readable without hiding the evidence chain.
