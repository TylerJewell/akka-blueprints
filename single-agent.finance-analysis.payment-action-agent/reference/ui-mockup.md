# UI mockup — antom-payment

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Antom Payment Action Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Antom <span class="accent">Payment Action Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Fill in the payment instruction form (type, currency, amount, recipient, method).
  3. Click **Submit instruction**.
  4. Watch the card move through AUTHORIZED → EXECUTING → SETTLED (or AWAITING_APPROVAL if above the high-value threshold).
- Card **How it works**: one paragraph on submit → authorize → optional HITL gate → agent execution → settle; one paragraph on the three governance mechanisms (before-tool-call guardrail, HITL gate, fraud-signal halt).
- Card **Components**: rows per component (PaymentEntity, FraudSignalConsumer, PaymentWorkflow, PaymentActionAgent, AuthorizationGuardrail, AntomApiSimulator, PaymentView, PaymentEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Simulator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `payment-card-data: true` declaration in Data is filled and prominent. `decisions.authority_level = autonomous-with-gate` and `oversight.human_in_loop = true` (for high-value) are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, H2). ID badges coloured: G1 red (guardrail), H1 amber (hitl), H2 orange (halt).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an instruction. <span class="accent">Watch it clear governance.</span>`. Subtitle: `One agent, three controls around it.`
- Layout: two-column.
  - **Left column** — Instruction panel + live list.
    - Instruction panel: dropdown `Payment type` (INITIATE / QUERY / REFUND), `Currency` text input (ISO 4217), `Amount` number input (major units displayed, converted to minor on submit), `Recipient ID` text input, dropdown `Method` (CARD / BANK_TRANSFER / WALLET), `Memo` text input, `Submitted by` text input, and a yellow `Submit instruction` button. A "Load seeded example" link fills all fields from `payment-instructions.jsonl`.
    - Live list below: one card per payment, newest-first. Each card shows status pill, payment type badge, amount (major units + currency), age. Cards in `AWAITING_APPROVAL` show a yellow "Approval pending" chip and inline Approve / Deny buttons.
  - **Right column** — Selected-payment detail.
    - Header: status pill + payment type badge + `paymentId`.
    - Instruction fields: type, currency, amount (major units), recipient id, method, memo, submitted by, submitted at.
    - Approval controls (visible only when status == AWAITING_APPROVAL): green **Approve** button + red **Deny** button.
    - Antom API response (visible after SETTLED / QUERIED): transaction id, Antom status code, settled amount + fee, currency, settled at timestamp.
    - Agent summary paragraph.
    - Fraud signal section (visible when fraudSignal != null): signal type chip + detail string + detected at timestamp. Card border highlights red.
- Status pill colours: REQUESTED=muted, AUTHORIZED=blue, REJECTED=red, AWAITING_APPROVAL=amber, EXECUTING=yellow, SETTLED=green, QUERIED=teal, HALTED=red, FAILED=red.
- Payment type badge colours: INITIATE=blue, QUERY=teal, REFUND=amber.
- Method badge colours: CARD=purple, BANK_TRANSFER=blue, WALLET=green.
