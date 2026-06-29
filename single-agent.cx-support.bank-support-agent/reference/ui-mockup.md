# UI mockup — bank-support-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Bank Support Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Bank Support <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded customer account (or enter a free-form `customerId`) and select an enquiry example (balance query / lost card / disputed transaction), or type your own.
  3. Click **Submit enquiry**.
  4. Watch the card transition through ACCOUNT_LOADED → RESPONDING → RESPONSE_RECORDED → LOGGED, and the risk badge and blockCard indicator appear.
- Card **How it works**: one paragraph on submit → lookup → respond → sanitize-log; one paragraph on the three governance mechanisms (before-tool-call guardrail, before-agent-response guardrail, PII sanitizer).
- Card **Components**: rows per component (EnquiryEntity, EnquiryWorkflow, BankSupportAgent, AccountLookupTool, ToolCallGuardrail, ResponseGuardrail, ResponseSanitizer, EnquiryView, EnquiryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 1 in-process Tool, 2 HttpEndpoints, plus 2 Guardrails as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Component-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` and `payment-card-data: true` declarations in Data are filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. The `card_block_requires_human_confirmation` field is `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic — a deliberate reminder that the deployer must decide whether card blocking requires a human confirmation step before execution.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, G2, S1). ID badges coloured: G1 and G2 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an enquiry. <span class="accent">Read the response.</span>`. Subtitle: `One agent, two guardrails, one sanitizer.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Customer account` (seeded accounts or free-form `customerId` input), dropdown `Category` (BALANCE_QUERY / DISPUTED_TRANSACTION / LOST_CARD / GENERAL), `Enquiry` textarea (with a "Load seeded example" link that fills the category and enquiry text), `Submitted by` text input, and a yellow `Submit enquiry` button.
    - Live list below: one card per enquiry, newest-first. Each card shows status pill, risk badge (1–10 coloured by range), blockCard indicator (lock icon when true), enquiry category chip, and age.
  - **Right column** — Selected-enquiry detail.
    - Header: status pill + risk badge + blockCard indicator + category chip + `customerId`.
    - Account summary card: masked account number, account type, available balance, card status (Active / Inactive).
    - Enquiry text: the submitted question.
    - Support response section: tone chip, answer paragraph, risk score bar (1–10 filled), blockCard recommendation (highlighted red when true).
    - Sanitized log section at bottom: a monospace block of the redacted answer with PII category chips above (`account-number`, `email`, `phone`, etc.). Label: "Audit log — PII redacted".
- Status pill colours: SUBMITTED=muted, ACCOUNT_LOADED=blue, RESPONDING=yellow, RESPONSE_RECORDED=blue, LOGGED=green, FAILED=red.
- Risk badge colours: 1–3=green, 4–6=yellow, 7–8=orange, 9–10=red.
- Tone chip colours: REASSURING=green, NEUTRAL=muted, URGENT=red.

The raw `response.answer` is never displayed in the UI — only the `sanitizedLog.redactedAnswer`. Support staff who need the raw answer for compliance review fetch `GET /api/enquiries/{id}` directly. This is intentional: the UI demonstrates that the model's output is audited in redacted form, even though the entity retains the raw text for authorized review.
