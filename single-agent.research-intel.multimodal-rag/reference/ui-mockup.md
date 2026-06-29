# UI mockup — multiformat-hybrid-rag

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: MultiformatHybridRAG</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Multiformat<span class="accent">HybridRAG</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded question (climate science / renewable energy policy / ocean biodiversity / carbon capture), or type your own.
  3. Click **Submit question**.
  4. Watch the card transition through RETRIEVING → ANSWERING → ANSWERED (or NO_RESULT).
- Card **How it works**: one paragraph on submit → retrieve → answer → citation-check; one paragraph on the citation guardrail.
- Card **Components**: rows per component (QueryEntity, ChunkIndexer, ChunkStore, QueryWorkflow, ResearchAgent, CitationGuardrail, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 ChunkStore as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled and prominent. `decisions.authority_level = informational` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; the expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the answer.</span>`. Subtitle: `One agent, one citation guardrail, mixed-format corpus.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded questions` (climate science / renewable energy policy / ocean biodiversity / carbon capture / custom), `Question` textarea (pre-filled when seeded option selected), `Submitted by` text input, and a yellow `Submit question` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, decision badge (ANSWERED / NO_RESULT / FAILED), question text truncated to 80 chars, age.
  - **Right column** — Selected-query detail.
    - Header: status pill + decision badge + question text.
    - Retrieved chunks section: a compact list of the top-K chunk cards, each showing chunkId, sourceTitle, format badge (TEXT / MARKDOWN / PDF_PAGE / IMAGE_CAPTION — each with its own colour), and the excerpt. Format badges: TEXT=muted, MARKDOWN=blue, PDF_PAGE=yellow, IMAGE_CAPTION=purple.
    - Answer narrative: the agent's 2–6-sentence paragraph when `ANSWERED`; a muted italic `noResultReason` when `NO_RESULT`.
    - Citation table (when ANSWERED): columns chunk id, source title, format badge, passage excerpt (italic). IMAGE_CAPTION rows carry a camera icon to indicate visual-source origin.
- Status pill colours: SUBMITTED=muted, RETRIEVING=blue, ANSWERING=yellow, ANSWERED=green, NO_RESULT=muted-orange, FAILED=red.
- Decision badge colours: ANSWERED=green, NO_RESULT=muted-orange, FAILED=red.
