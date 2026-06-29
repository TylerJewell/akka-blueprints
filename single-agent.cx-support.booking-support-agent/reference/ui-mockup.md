# UI mockup — booking-support-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Customer Support Booking Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Customer Support <span class="accent">Booking Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Select a customer id (C001 / C002 / C003) and type a message, or pick one of the four seeded scenarios.
  3. Click **Send**.
  4. Watch the turn transition through SANITIZED → AGENT_REPLIED and inspect the tool-call log and updated booking.
- Card **How it works**: one paragraph on receive message → sanitize → agent turn → record; one paragraph on the three governance mechanisms (PII sanitizer, before-tool-call guardrail, CI judge gate).
- Card **Components**: rows per component (BookingSessionEntity, PiiSanitizer, BookingSessionWorkflow, BookingSupportAgent, ToolCallGuardrail, BookingStore, BookingSessionView, SupportEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Store as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Component-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` and `payment-card-data: true` declarations in Data are filled and prominent. `decisions.authority_level = full-execution` is flagged as a high-attention field. `oversight.human_in_loop = false` and `human_on_loop = true` are the distinctive oversight answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, S1, H1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer), H1 purple (ci-gate).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask. Change. <span class="accent">Confirm.</span>`. Subtitle: `One agent, one guardrail, one sanitizer.`
- Layout: two-column.
  - **Left column** — Session panel.
    - Customer selector: dropdown (C001 / C002 / C003) or free-text input.
    - Scenario picker: dropdown of four seeded scenarios (look up flight, change seat, cancel booking, ask about refund policy) that pre-fills the message textarea.
    - Message textarea + **Send** button (yellow).
    - Scrollable turn history below: one card per turn, newest at top. Each card shows status pill, turn age, and a one-line preview of the sanitized message.
  - **Right column** — Selected-turn detail.
    - Header: status pill + `customerId` + turn timestamp.
    - Sanitized message block: monospace text with PII-category chips above (e.g., `payment-card`, `email`). An info note: "Raw message retained in audit log; this is what the agent saw."
    - Tool-call log: a small table with columns tool name, booking ref, requested change, outcome chip (green ALLOWED / red BLOCKED), guardrail reason (shown when BLOCKED).
    - Agent reply: the `replyText` paragraph.
    - Updated booking card (shown when `updatedBooking` is non-null): fields flight number, origin → destination, departure, seat, new status.
- Status pill colours: RECEIVED=muted, SANITIZED=blue, AGENT_REPLIED=green, FAILED=red.
- Tool outcome chip colours: ALLOWED=green, BLOCKED=red.
- BookingStatus chip colours: CONFIRMED=green, MODIFIED=yellow, CANCELLED=red, REFUND_PENDING=yellow.
