# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark, yellow accent, Instrument Sans, dot-grid background. Content wraps in a 1080px column with no horizontal scroll (Lesson 12).

Browser title: `<title>Akka Sample: High-Volume Document Analyzer</title>`.

Five left-nav tabs. Each `.nav-tab` carries `data-tab="N"`; each `.tab-panel` carries `data-panel="N"`.

## Tab 1 — Overview

Eyebrow "Overview" + headline "High-Volume Document Analyzer", **no subtitle**. Four cards:

- **Try it** — the single command `/akka:build`. No env-var export block.
- **How it works** — three lines: partition each document into extraction + classification work items, run Extractor and Classifier in parallel, merge and sanitize PII before persisting.
- **Components** — the count: 3 autonomous-agent, 1 workflow, 3 event-sourced-entity, 1 view, 1 consumer, 2 timed-action, 2 http-endpoint.
- **API contract** — the top-level endpoints from `api-contract.md`.

## Tab 2 — Architecture

The four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER) plus a click-to-expand component table with syntax-highlighted Java snippets. Carry the Lesson 24 CSS overrides: white `fill`/`color` on every state-label DOM path, `overflow:visible` on edge-label `foreignObject`, and `transitionLabelColor:#cccccc` in `mermaid.initialize`.

## Tab 3 — Risk Survey

Renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style from `governance.html`. Question on the left (yellow label), answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Renders `/api/metadata/eval-matrix` in `matrix-card` / `matrix-row` style. The label column carries the control id and a colored mechanism pill (`sanitizer` orange, `eval-event` blue). Click a row to expand name, rationale, and implementation. Two controls: S1, E1.

## Tab 5 — App UI

A form with a multi-line `rawDocuments` textarea (one document per line) and a `submittedBy` text input and a Submit button (`POST /api/documents/batches`). Below it, a live list of documents streamed from `/api/documents/sse`. Each row shows the reference number (if extracted), a status pill (QUEUED / PROCESSING / PROCESSED / DEGRADED), the risk category pill (LOW / MEDIUM / HIGH / CRITICAL), and the quality score when present. Click a row to expand the extracted fields, the classification with flagged topics, the sanitized text, the PII redaction list, and the quality rationale.

A separate batch-progress bar at the top of the document list shows the current batch's `processedCount / totalDocuments` and the batch status pill (PENDING / IN_PROGRESS / COMPLETE / PARTIAL).

## Tab switching MUST be attribute-based (Lesson 26)

Switch by `data-tab` / `data-panel` attribute, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Exactly five `.tab-panel` sections exist in the DOM. Removed tabs are deleted, never hidden with `display:none` — a hidden zombie panel takes an index slot and blanks the App UI tab.
