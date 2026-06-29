# UI mockup — basic-rag-pipeline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Basic RAG Workflow</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Basic RAG <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded questions (or type your own) and click **Ask**.
  3. Watch the card transition through INDEXING → INDEXED → ANSWERING → ANSWERED.
  4. Inspect the guardrail chip on the card — PASSED means every citation was verified against the indexed corpus.
- Card **How it works**: one paragraph on the two task phases (INGEST → QUERY) and the typed handoff between them (the `IndexedCorpus` produced by INGEST is the context for QUERY); one paragraph on the governance mechanism (citation-grounding output guardrail, before-agent-response hook).
- Card **Components**: rows per component (RagSessionEntity, RagPipelineWorkflow, RagAgent, IngestTools, QueryTools, VectorStore, AnswerGuardrail, RagSessionView, RagEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 2 function-tool classes, 1 VectorStore, and 1 Guardrail as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the question is the only user input; the corpus is an in-process document set). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the grounded answer.</span>`. Subtitle: `One agent, two task phases, one citation check between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Question` (with a "Pick a seeded question" dropdown that fills it), and a yellow `Ask` button.
    - Live list below: one card per session, newest-first. Each card shows status pill, guardrail chip (PASSED/BLOCKED when determined), question excerpt, and age.
  - **Right column** — Selected-session detail.
    - Header: status pill + guardrail chip + question.
    - Phase panel 1 (Indexed corpus): chunk count and a compact table of document titles. Visible once `corpus.isPresent()`.
    - Phase panel 2 (Answer): answer text paragraph followed by a citation list, each citation showing the chunk excerpt and source URL as a chip. Visible once `answer.isPresent()`.
    - Guardrail section at bottom: verdict badge (green PASSED / red BLOCKED) and the one-line reason. Always visible once `guardrailOutcome.isPresent()`.
- Status pill colours: CREATED=muted, INDEXING=blue, INDEXED=blue, ANSWERING=yellow, ANSWERED=green, BLOCKED=red, FAILED=red.
- Guardrail chip colours: PASSED=green, BLOCKED=red.

Each phase panel renders only when its data is present on the row record. A session in `ANSWERING` shows panel 1 (corpus indexed) and panel 2 with a spinner. This is the visual proof that the typed handoff between phases is the only path information travels.
