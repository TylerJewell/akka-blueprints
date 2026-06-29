# UI mockup — restaurant-booking-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SK Restaurant Booking</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `SK Restaurant <span class="accent">Booking</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded booking request (Bella Napoli / Le Petit Bistro / Smoke & Ember) or type your own.
  3. Click **Start booking** and watch the card reach `PENDING_CONFIRMATION`.
  4. Click **Confirm** to commit the reservation or **Decline** to cancel.
- Card **How it works**: one paragraph on request → collect → confirm → commit; one paragraph on the two governance mechanisms (confirmation gate, before-tool-call guardrail).
- Card **Components**: rows per component (BookingEntity, ConfirmationConsumer, BookingWorkflow, BookingAgent, ReservationGuardrail, BookingStub, BookingView, BookingEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Stub as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `pii: true` (contact name and email) is filled and prominent. `decisions.authority_level = automated-action` and `oversight.human_in_loop = true` are the distinctive answers. `decisions.human_confirmation_required_before_external_write = true` is highlighted. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, G1). ID badges coloured: H1 amber (hitl), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Request a table. <span class="accent">Confirm before it's booked.</span>`. Subtitle: `One agent, two governance layers, one reservation.`
- Layout: two-column.
  - **Left column** — Request panel + live list.
    - Request panel: dropdown `Example requests` (Bella Napoli / Le Petit Bistro / Smoke & Ember / custom), `Your request` textarea (with a "Load example" link that fills the textarea), `Requested by` text input, and a yellow `Start booking` button.
    - Live list below: one card per booking, newest-first. Each card shows status pill, restaurant name (once the proposal lands), age. Cards in `PENDING_CONFIRMATION` state show a pulsing amber border.
  - **Right column** — Selected booking detail.
    - Header: status pill + booking id.
    - Raw request: the user's original freeform text in a monospace block.
    - Proposal summary (appears once `PENDING_CONFIRMATION` is reached): restaurant name, date, time, party size, contact name and email, estimated total. Two action buttons: green `Confirm` and red `Decline`.
    - Confirmation chip (appears in `CONFIRMED` state): green badge with the confirmation number and confirmed-at timestamp.
    - Failure section (appears in `FAILED` or `DECLINED` state): the failure reason or "Booking declined by user" in a red block.
- Status pill colours: INITIATED=muted, COLLECTING=yellow, PENDING_CONFIRMATION=amber, COMMITTING=blue, CONFIRMED=green, DECLINED=muted, FAILED=red.
- Confirm/Decline buttons are only rendered when `status === "PENDING_CONFIRMATION"`. They are not rendered in any other state.
