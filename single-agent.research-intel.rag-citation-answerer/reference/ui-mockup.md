# UI mockup — rag-citation-answerer

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: RAG Citation Answerer</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `RAG Citation <span class="accent">Answerer</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Upload a seeded document (climate-policy brief, public-health summary, or software-procurement note) or paste your own.
  3. Wait for the document status to reach `INDEXED`, then type a question and click **Ask**.
  4. Watch the query card transition through `RETRIEVING` → `ANSWERED`, then inspect the citations.
- Card **How it works**: one paragraph on upload → sanitize → chunk → index; one paragraph on question → retrieve → answer → guardrail-validate.
- Card **Components**: rows per component (DocumentEntity, QueryEntity, DocumentSanitizer, ChunkIndexer, ChunkStore, AnswerWorkflow, CitationAnswererAgent, CitationGuardrail, DocumentView, QueryView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequences, state machines, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 2 Views, 2 Consumers, 2 HttpEndpoints, plus 1 Guardrail + 1 Store as supporting classes).
- Mermaid cards in order: (a) component graph, (b) interaction sequence, (c) DocumentEntity state machine, (d) QueryEntity state machine, (e) entity model. All render with Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` and `pii_handled_by_sanitizer_before_llm: true` declarations in Data are filled and prominent. `decisions.authority_level = informational` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Upload documents. <span class="accent">Ask questions.</span>`. Subtitle: `One agent, grounded citations, two governance controls.`
- Layout: two-column.
  - **Left column** — Upload panel + Document list + Ask panel + Query list.
    - **Upload panel**: `Document title` text input, `Document` textarea (with a "Load seeded example" link that populates both fields from one of the three seed documents), `Uploaded by` text input, and a `Upload document` button.
    - **Document list** below: one card per document, newest-first. Each card shows status pill, document title, chunk count (once indexed), and age.
    - **Ask panel**: `Question` textarea, a multi-select checkbox list of indexed documents (grayed-out until at least one document reaches `INDEXED`), `Asked by` text input, and an `Ask` button.
    - **Query list** below: one card per query, newest-first. Each card shows status pill, confidence badge (once answered), truncated question text, and age.
  - **Right column** — Selected query detail + selected document detail.
    - **Query detail**: header with status pill, confidence badge, and truncated question. Answer text paragraph. Citations table: columns chunkId, document title, section hint, excerpt (italic). Hovering a citation row highlights the matching chunkId mention in the answer text.
    - **Document detail** (shown when a document card is selected instead): document title, status pill, chunk count, PII category chips, sanitized text preview in a monospace block.
- Status pill colours: UPLOADED=muted, SANITIZED=blue, CHUNKED=blue, INDEXED=green, FAILED=red (documents); SUBMITTED=muted, RETRIEVING=yellow, ANSWERED=green, FAILED=red (queries).
- Confidence badge colours: HIGH=green, MEDIUM=yellow, LOW=muted.
- The raw document text is never displayed in the UI — only the sanitized form. A user who needs the raw text fetches `GET /api/documents/{id}` directly. This is intentional.
- The Ask button is disabled if no indexed documents are selected and the question textarea is empty. Client-side validation only — server-side accepts the request and returns an answer with `confidence = LOW` and empty citations if no chunks are found.
