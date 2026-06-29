# UI mockup — schema-extractor

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SchemaExtractor</title>`.

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
- Headline: `Schema<span class="accent">Extractor</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded schema (Invoice / PurchaseOrder / SupportTicket) and load the matching seed document, or paste your own.
  3. Click **Submit for extraction**.
  4. Watch the card transition through SANITIZED → EXTRACTING → RECORD_EXTRACTED.
- Card **How it works**: one paragraph on submit → sanitize → extract; one paragraph on the two governance mechanisms (PII sanitizer, schema-output guardrail).
- Card **Components**: rows per component (ExtractionJobEntity, DocumentSanitizer, ExtractionWorkflow, ExtractionAgent, RecordGuardrail, ExtractionView, ExtractionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = automated-action` and `oversight.human_in_loop = false` are the distinctive answers — this is a fully automated pipeline, not an advisory tool. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a document. <span class="accent">Get a record.</span>`. Subtitle: `One agent, schema-bound output, PII never seen by the model.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Target schema` (Invoice / PurchaseOrder / SupportTicket / custom), `Document title` text input, `Document` textarea (with a "Load seeded example" link that fills both title and body from the matching schema's seed document), `Submitted by` text input, and a yellow `Submit for extraction` button.
    - Live list below: one card per job, newest-first. Each card shows status pill, schema-coverage badge (when record landed), document title, age.
  - **Right column** — Selected-job detail.
    - Header: status pill + schema-coverage badge + `documentTitle`.
    - Target schema fields: a compact table listing `fieldName`, `fieldType` chip, `required` indicator, `maxLength`.
    - Sanitized document preview: a monospace block of the redacted document, with PII category chips above (`email`, `phone`, `address`, etc.).
    - Extracted fields table: columns field name, type chip, value, confidence chip (HIGH=green, MEDIUM=blue, LOW=yellow, ABSENT=muted).
    - Schema coverage bar: a progress bar showing `schemaCoveragePercent` with the numeric label. Coverage below 80% highlights the bar amber; below 50% highlights red.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, EXTRACTING=yellow, RECORD_EXTRACTED=green, FAILED=red.
- Confidence chip colours: HIGH=green, MEDIUM=blue, LOW=yellow, ABSENT=muted.
- Schema-coverage badge colours: 100%=green, 80–99%=blue, 50–79%=amber, <50%=red.

The raw document is never displayed on this screen — only the sanitized form. Operators who need the raw text fetch `/api/jobs/{id}` and read `request.rawDocument` from the JSON.
