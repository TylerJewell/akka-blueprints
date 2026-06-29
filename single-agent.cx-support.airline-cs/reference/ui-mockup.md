# UI mockup — airline-cs

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: AirlineCS</title>`.

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
- Headline: `Airline<span class="accent">CS</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded request example (seat change / flight rebook / complaint) and load it with one click.
  3. Click **Submit request**.
  4. Watch the card transition through SANITIZED → PROCESSING → AWAITING_CONFIRMATION → COMPLETED (or CANCELLED).
- Card **How it works**: one paragraph on receive → sanitize → ReAct loop → HITL gate → complete; one paragraph on the three governance mechanisms (PII sanitizer, before-tool-call guardrail, application HITL gate).
- Card **Components**: rows per component (RequestEntity, MessageSanitizer, ServiceWorkflow, CustomerServiceAgent, ModificationGuardrail, BookingService, RequestView, ServiceEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, 1 Guardrail + 1 supporting service class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data and `customer_confirmation_required_for_writes: true` in Oversight are prominent. `decisions.authority_level = automated-with-confirmation` is the distinctive answer. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` appear faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, H1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail), H1 orange (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a request. <span class="accent">Confirm the change.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Request type` (Seat change / Flight rebook / Complaint / Custom), `Booking reference` text input (optional), `Message` textarea (with a "Load seeded example" link that fills both booking ref and message), `Submitted by` text input, and a yellow `Submit request` button.
    - Live list below: one card per request, newest-first. Each card shows status pill, outcome badge (when completed), request type chip, booking ref, age.
  - **Right column** — Selected-request detail.
    - Header: status pill + outcome badge + `bookingRef`.
    - Sanitized message preview: the redacted text, with PII category chips above (`email`, `phone`, `ffp`, etc.).
    - Tool call log: a collapsible list of tool calls with tool name, arguments summary, result summary, and timestamp. Shown in call order.
    - Confirmation panel (visible only when status is `AWAITING_CONFIRMATION`): a yellow highlighted box showing the proposed change text, a green **Confirm** button, and a grey **Cancel** button. On click, POSTs to `/api/requests/{id}/confirmation`.
    - Outcome section (visible when `COMPLETED` or `CANCELLED`): outcome status badge, summary paragraph, `caseNumber` badge (when non-null).
- Status pill colours: RECEIVED=muted, SANITIZED=blue, PROCESSING=yellow, AWAITING_CONFIRMATION=orange, COMPLETED=green, CANCELLED=muted, FAILED=red.
- Outcome badge colours: RESOLVED=green, NEEDS_FOLLOWUP=yellow, CANCELLED=muted, FAILED=red.
- The raw message is never displayed — only the sanitized form. Reviewers who need the raw text fetch `/api/requests/{id}` and read `request.rawMessage` from the JSON.
