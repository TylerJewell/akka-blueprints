# UI mockup — invoice-processing

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Invoice Processing</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index:

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
- Headline: `Invoice <span class="accent">Processing</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Select a seeded invoice from the vendor dropdown (or paste raw invoice text) and click **Submit invoice**.
  3. Watch the card transition through EXTRACTING → EXTRACTED → VALIDATING → VALIDATED → POSTING → POSTED → EVALUATED (or pause at APPROVAL_REQUESTED for high-value invoices).
  4. For an APPROVAL_REQUESTED card, click **Approve** or **Deny** in the right pane to resume the workflow.
- Card **How it works**: one paragraph on the three task phases (EXTRACT → VALIDATE → POST) and the typed handoff between them; one paragraph on the two governance mechanisms (PII sanitizer at the extract→validate boundary, HITL approval gate for high-value invoices).
- Card **Components**: rows per component (InvoiceEntity, InvoicePipelineWorkflow, InvoiceAgent, ExtractTools, ValidateTools, PostTools, PhaseGuardrail, VendorPiiSanitizer, PostingQualityScorer, InvoiceView, InvoiceEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, 1 Sanitizer, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and notes that vendor contact fields are redacted by S1. `decisions.authority_level = execute` (below threshold, posting is fully automated) and `oversight.human_in_loop = true` (H1 gate for high-value) are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, H1). ID badges coloured: S1 blue (sanitizer), H1 orange (HITL).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an invoice. <span class="accent">Track the posting.</span>`. Subtitle: `One agent, three task phases, PII redacted at the boundary, high-value amounts reviewed by a human.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a vendor dropdown (`Pick a seeded invoice`) that populates a read-only raw-text preview, plus a **Submit invoice** button.
    - Live list below: one card per invoice, newest-first. Each card shows status pill, eval score chip (when eval landed), vendor name, invoice number, total amount, and age. High-value cards paused at APPROVAL_REQUESTED display an orange "Awaiting approval" badge.
  - **Right column** — Selected-invoice detail.
    - Header: status pill + eval score chip + vendor name + invoice number + total amount.
    - Sanitization badge: a small green lock icon with the label "PII redacted" and the list of redacted field names. Always visible once `sanitization.isPresent()`.
    - Phase panel 1 (Extracted invoice): a header table (invoiceNumber, vendorName, vendorEmail=[REDACTED], vendorPhone=[REDACTED], invoiceDate, currency, totalAmount) and a line-item table (lineId, description, quantity, unitPrice, lineTotal). Visible once `extracted.isPresent()`.
    - Phase panel 2 (Validated line items): a table with columns lineId, description, lineTotal, GL account code, GL account name, balanceOk. Plus a balance summary row. Visible once `validated.isPresent()`.
    - Approval panel (only visible when `status == APPROVAL_REQUESTED` or `APPROVED` or `DENIED`): shows total amount and threshold; renders Approve / Deny buttons when status is `APPROVAL_REQUESTED`. Approve calls `POST /api/invoices/{id}/approve`; Deny calls `POST /api/invoices/{id}/deny`.
    - Phase panel 3 (Journal entry): a table of journal lines (accountCode, accountName, debit, credit) with a totals row confirming balance. Confirmation reference shown below. Visible once `posted.isPresent()`.
    - Eval section at bottom: a 1–5 score widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: CREATED=muted, EXTRACTING=blue, EXTRACTED=blue, VALIDATING=yellow, VALIDATED=yellow, APPROVAL_REQUESTED=orange, APPROVED=yellow, POSTING=blue, POSTED=blue, EVALUATED=green, DENIED=muted, FAILED=red.

Each phase panel renders only when its data is present on the row record. A card in `VALIDATING` shows panels 1 and 2 (panel 2 with a spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels — the validated panel cannot appear before the extract panel.
