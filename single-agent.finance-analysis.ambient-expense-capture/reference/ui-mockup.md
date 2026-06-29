# UI mockup — ambient-expense-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Ambient Expense Capture</title>`.

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
- Headline: `Ambient Expense <span class="accent">Capture</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Load a seeded receipt (restaurant / hotel folio / travel itinerary) or paste your own receipt text.
  3. Click **Submit receipt**.
  4. Watch the card transition through SANITIZED → CAPTURING → REPORT_READY → SUBMITTED_TO_SYSTEM.
- Card **How it works**: one paragraph on submit → sanitize → capture → submit; one paragraph on the two governance mechanisms (PII sanitizer, before-tool-call guardrail).
- Card **Components**: rows per component (ExpenseEntity, ReceiptSanitizer, ExpenseWorkflow, ExpenseCaptureAgent, SubmissionGuardrail, ExpenseView, ExpenseEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` and `payment-card-data: true` declarations in Data are filled and prominent. `decisions.authority_level = autonomous` and `oversight.human_reviews_blocked_items = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a receipt. <span class="accent">Get a categorized report.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: `Receipt text` textarea (with a "Load seeded example" dropdown — Restaurant / Hotel Folio / Travel Itinerary — that fills the textarea), `Submitter ID` text input, optional `Trip or project code` text input, and a yellow `Submit receipt` button.
    - Live list below: one card per submission, newest-first. Each card shows status pill, report-status badge (when report lands), document title derived from the first merchant name or "Receipt", age.
  - **Right column** — Selected-submission detail.
    - Header: status pill + report-status badge + submitter ID + trip code (if present).
    - Sanitized receipt preview: a monospace block of the redacted text, with PII category chips above (`payment-card-number`, `person-name`, `address`, etc.).
    - Line-item table: columns description, amount, currency, category, merchant, date, status (coloured chip), block reason (italic, shown only when BLOCKED).
    - Total row at table bottom: sum of all line items.
    - Report status badge: `FULLY_APPROVED` (green) / `PARTIALLY_BLOCKED` (yellow) / `FULLY_BLOCKED` (red).
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, CAPTURING=yellow, REPORT_READY=blue, SUBMITTED_TO_SYSTEM=green, FAILED=red.
- Report status badge colours: FULLY_APPROVED=green, PARTIALLY_BLOCKED=yellow, FULLY_BLOCKED=red.
- Line-item status chip colours: APPROVED=green, FLAGGED=yellow, BLOCKED=red.

The raw receipt text is never displayed on this screen — only the sanitized form. Submitters who need the raw text fetch `/api/expenses/{id}` and read `request.rawReceiptText` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the original.
