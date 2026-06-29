# UI mockup — whatsapp-fintech-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: WhatsApp Fintech Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. Canonical implementation:

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
- Headline: `WhatsApp <span class="accent">Fintech Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded account and paste a seed message (balance query / small transfer / large transfer), or click **Load example** to fill both.
  3. Click **Send message**.
  4. Watch the card transition through SANITIZED → RESPONDING → RESPONSE_READY → (AWAITING_APPROVAL →) EXECUTED.
- Card **How it works**: one paragraph on receive → sanitize → respond → HITL-gate → execute; one paragraph on the three governance mechanisms (PII sanitizer, before-tool-call guardrail, application HITL gate).
- Card **Components**: rows per component (MessageEntity, MessageSanitizer, MessageWorkflow, FintechQueryAgent, PaymentGuardrail, HitlGate, MessageView, MessageEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 HitlGate as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed component-row table with the Java file path per component (matches `PLAN.md` component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = autonomous` combined with `hitl_threshold_usd = 500` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` appear faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, H1). ID badge colours: S1 green (sanitizer), G1 red (guardrail), H1 purple (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Watch the approval gate.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Account` (seeded customer accounts), `Message` textarea (with a **Load example** link that fills the textarea with a seed message), and a teal `Send message` button.
    - Live list below: one card per message, newest-first. Each card shows status pill, intent badge (when response landed), document title (first 60 chars of raw text), age. Cards in `AWAITING_APPROVAL` show a yellow pulsing border.
  - **Right column** — Selected-message detail.
    - Header: status pill + intent badge + message age.
    - Inbound text: a short excerpt of the raw message text (first 120 chars).
    - Sanitized text preview: a monospace block of the redacted message, with PII category chips above (`phone`, `account-number`, etc.).
    - Agent response section: `responseText` paragraph, collapsible tool-calls table (tool name, args JSON, result).
    - Pending transaction panel (shown only when `pendingTransaction` is present): a data card with from/to accounts, amount, currency, description. When status is `AWAITING_APPROVAL`, two large buttons: **Approve** (teal) and **Reject** (red), each prompting for an operator note before submitting.
    - HITL decision record (shown when `hitlDecision` is present): outcome badge (APPROVED=green, REJECTED=red), operator ID, note, timestamp.
- Status pill colours: RECEIVED=muted, SANITIZED=blue, RESPONDING=yellow, RESPONSE_READY=blue, AWAITING_APPROVAL=orange, EXECUTED=green, HITL_REJECTED=red, FAILED=red.
- Intent badge colours: BALANCE_QUERY=blue, FUND_TRANSFER=orange, TRANSACTION_HISTORY=teal, ACCOUNT_INFO=muted, UNKNOWN=red.

The raw message text is shown only as a short excerpt in the detail header — not the full text. The full raw text is accessible via `GET /api/messages/{id}` at `inbound.rawText`. The sanitized form is what the UI displays prominently, demonstrating that the model's input is the redacted form even though the audit trail keeps the original.
