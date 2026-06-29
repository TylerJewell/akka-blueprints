# UI mockup — pgvector-rag-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: RAG Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `RAG <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Optionally ingest a document via the Ingest panel, or use the three seeded corpus entries.
  3. Type a question in the Question field and click **Ask**.
  4. Watch the card transition through RETRIEVING → ANSWERING → ANSWERED → EVALUATED.
- Card **How it works**: one paragraph on question → embed → retrieve → answer → eval; one paragraph on the two governance mechanisms (citation guardrail, on-decision eval).
- Card **Components**: rows per component (QuestionEntity, CorpusEntity, CorpusIngestionConsumer, QueryWorkflow, CorpusQueryAgent, CitationGuardrail, AnswerEvaluationScorer, QuestionView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes, 1 external pgvector store).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `decisions.authority_level = informational` declaration and `oversight.human_in_loop = true` are the distinctive answers. The `corpus-documents-licensed-for-ai-processing` field is `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic — this is the key deployer-specific declaration for a corpus-based system.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Get a grounded answer.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Question submission panel + corpus ingest panel + live list.
    - Question submission panel: `Question` textarea (placeholder: "How does emptyState() work?"), `Asked by` text input, and a yellow `Ask` button. A "Load seeded question" link fills the textarea with a random seed question.
    - Corpus ingest panel (collapsible, closed by default): `Source label` text input, `Document body` textarea, and an `Ingest document` button. Shows a compact list of indexed documents (source label, chunk count, status pill) below.
    - Live list below: one card per question, newest-first. Each card shows status pill, eval score chip (when eval landed), a truncated version of the question text, and age.
  - **Right column** — Selected-question detail.
    - Header: status pill + eval score chip + full question text.
    - Retrieved passages section: a table of the passages the workflow retrieved (chunk id, source label, relevance score badge, first 120 chars of text). Relevance score > 0.7 rendered in green, 0.5–0.7 in yellow, < 0.5 in muted.
    - Answer section: the answer text rendered with `[chunkId]` markers displayed as clickable inline chips. Clicking a chip highlights the corresponding row in the passages table.
    - Citations list: a compact list of `chunkId` + `sourceLabel` pairs.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, RETRIEVING=blue, ANSWERING=yellow, ANSWERED=blue, EVALUATED=green, FAILED=red.
- Relevance score thresholds: ≥ 0.7 = green badge, 0.5–0.69 = yellow badge, < 0.5 = muted badge.

The raw document body stored on `CorpusEntity` is never displayed in the main UI — only the chunk count and source label. Developers who need the raw body fetch `/api/corpus/{id}` directly. This mirrors the pattern of keeping large payloads off the hot path.
