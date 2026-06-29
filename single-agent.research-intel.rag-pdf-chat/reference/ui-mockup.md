# UI mockup — rag-pdf-chat

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Chat With PDF Docs Using AI (Quoting Sources)</title>`.

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
- Headline: `PDF Chat with <span class="accent">Cited Sources</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **Load seeded PDF** to upload one of the three example documents.
  3. Type a question into the chat input and click **Ask**.
  4. Watch the exchange card transition through RETRIEVING → ANSWERING → ANSWERED, then click a citation marker to jump to the source passage.
- Card **How it works**: one paragraph on upload → index → retrieve → answer; one paragraph on the guardrail mechanism (citation-validator, before-agent-response).
- Card **Components**: rows per component (PdfDocumentEntity, PassageRetriever, ChatSessionWorkflow, PdfChatAgent, CitationGuardrail, ChatSessionView, ChatEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machines, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, document state machine, exchange state machine) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `user_expected_to_verify_citations: true` and `documents_may_contain_sensitive_content: TO_BE_COMPLETED_BY_DEPLOYER` declarations are prominent. Many deployer fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the cited answer.</span>`. Subtitle: `One agent, one citation guardrail around it.`
- Layout: two-column.
  - **Left column** — Upload panel + document list + chat panel.
    - Upload panel: a **Load seeded PDF** dropdown (3 options: distributed-systems-whitepaper, data-governance-policy-brief, rag-research-article), an **Upload PDF** file picker, and a `Session ID` text input (auto-filled with a random id).
    - Document list: one row per indexed document, showing status chip, filename, passage count, and age.
    - Chat panel (visible once a document is selected): a `Question` textarea and a yellow `Ask` button. Below: the exchange history list, newest-first. Each exchange card shows the question text, status chip, and (once answered) the answer text with inline `[P-xxx]` citation markers rendered as clickable chips.
  - **Right column** — Citations panel.
    - Header: selected exchange's question text + status chip.
    - Retrieved passages list: one card per retrieved passage, each showing passageId chip, page number chip, and passage text. Cards for passages that were actually cited have a highlighted border.
    - When an exchange is `UNANSWERABLE`, the Citations panel shows a single message card: "No passages were cited. The document does not contain an answer to this question."
- Status chip colours: RETRIEVING=yellow, ANSWERING=yellow, ANSWERED=green, UNANSWERABLE=muted, FAILED=red.
- Citation marker chips: coloured teal, clickable; clicking scrolls the matching passage card into view in the Citations panel.
- Document status chip colours: UPLOADING=muted, INDEXED=green, FAILED=red.
