# UI mockup — medical-document-pipeline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Medical Document Processing Assistant</title>`.

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
- Headline: `Medical Document <span class="accent">Processing Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded documents (or paste your own masked text) and click **Process document**.
  3. Watch the card transition through SANITIZING → SANITIZED → EXTRACTING → EXTRACTED → AWAITING_REVIEW.
  4. Review the extracted fields in the right pane, approve or correct them, and click **Approve**. Watch the card reach SUMMARIZING → SUMMARIZED → EVALUATED.
- Card **How it works**: one paragraph on the three task phases (EXTRACT → [clinician review] → WRITE_SUMMARY) and the typed handoff between them; one paragraph on the four governance mechanisms (PHI sanitizer, PII sanitizer, human-in-the-loop gate, clinical accuracy eval).
- Card **Components**: rows per component (DocumentEntity, DocumentPipelineWorkflow, MedicalDocumentAgent, ExtractTools, ValidateTools, SummarizeTools, SanitizationGuardrail, ClinicalAccuracyScorer, DocumentView, DocumentEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` and `phi: true` declarations in Data are filled and highlighted (the blueprint explicitly handles both). `decisions.authority_level = clinical-documentation-support` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with four rows (S1, S2, H1, E1). ID badges coloured: S1 and S2 amber (sanitizer), H1 orange (hitl), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Upload a document. <span class="accent">Get a clinical summary.</span>`. Subtitle: `One agent, three task phases, a clinician gate between extraction and summarization.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a `Document type` dropdown (`discharge-summary` / `radiology-report` / `referral-letter`), a text area for document text (seeded with a "Pick a sample" button that fills it with the selected sample document), and a yellow `Process document` button.
    - Live list below: one card per document, newest-first. Each card shows status pill, eval score chip (when eval landed), document type, age, and a small red dot if any guardrail rejection fired. Cards in `AWAITING_REVIEW` show a pulsing amber indicator to draw the clinician's attention.
  - **Right column** — Selected-document detail.
    - Header: status pill + eval score chip + document type.
    - Phase panel 1 (Masked text preview): a scrollable pre block showing the first 500 characters of `maskedDocument.maskedText` with token spans highlighted. Visible once `maskedDocument` is present.
    - Phase panel 2 (Extracted fields): four sub-sections (Demographics, Diagnoses, Medications, Vitals), each a table with columns field name, extracted value, confidence bar. Visible once `extractedFields` is present.
    - Clinician review form (only visible when `status === "AWAITING_REVIEW"`): same table as Phase panel 2 but with editable value cells, plus an **Approve** button and a **Submit corrections** button. Submitting calls `POST /api/documents/{id}/review`.
    - Phase panel 3 (Validation findings): a table with columns field name, severity badge (OK/WARNING/ERROR), message. Visible once `validationResult` is present.
    - Phase panel 4 (Clinical summary): document type header, patient context block, chief complaint, per-section blocks (heading, body, source field chips), clinical impression, recommendations. Visible once `clinicalSummary` is present.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if `guardrailRejections` is non-empty): a small table with phase, tool, reason, time.
- Status pill colours: RECEIVED=muted, SANITIZING=blue, SANITIZED=blue, EXTRACTING=yellow, EXTRACTED=yellow, AWAITING_REVIEW=amber, REVIEW_SUBMITTED=amber, SUMMARIZING=blue, SUMMARIZED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A document in `AWAITING_REVIEW` shows panels 1, 2, and the review form; panels 3 and 4 are hidden until the downstream steps complete. This is the visual proof that the typed handoff between phases is the only path information travels, and that the clinician gate is a real pause — not cosmetic.
