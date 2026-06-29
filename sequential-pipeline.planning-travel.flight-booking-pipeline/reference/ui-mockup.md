# UI mockup — flight-booking-pipeline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Flight Booking</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Flight <span class="accent">Booking</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded routes (or fill in your own origin / destination / date) and click **Find flights**.
  3. Watch the card transition through SEARCHING → SEARCH_DONE → SELECTING → SEAT_SELECTED → AWAITING_CONFIRMATION.
  4. Review the itinerary in the right pane and click **Confirm booking** to commit the reservation.
- Card **How it works**: one paragraph on the three task phases (SEARCH → SELECT → BOOK) and the typed handoff between them; one paragraph on the two governance mechanisms (user confirmation gate, phase-gate guardrail).
- Card **Components**: rows per component (BookingEntity, BookingPipelineWorkflow, BookingAgent, SearchTools, SelectTools, BookTools, BookingGuardrail, BookingView, BookingEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = action-on-behalf-of-user` and `oversight.human_in_loop = true` declarations are the distinctive answers — this agent commits real spend, not just advice. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, G1). ID badges coloured: H1 teal (HITL), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Search a route. <span class="accent">Confirm before it books.</span>`. Subtitle: `One agent, three task phases, one human approval gate between selection and commitment.`
- Layout: two-column.
  - **Left column** — Search panel + live list.
    - Search panel: three inputs — `Origin` (e.g. SFO), `Destination` (e.g. JFK), `Travel date` (date picker) — plus a "Pick a seeded route" dropdown that fills all three, and a yellow `Find flights` button.
    - Live list below: one card per booking, newest-first. Each card shows status pill, route summary (SFO → JFK · Sep 15), age, and a small red dot if any guardrail rejection fired during this booking.
  - **Right column** — Selected-booking detail.
    - Header: status pill + route.
    - Phase panel 1 (Flight offers): a table with columns carrier, flight number, departure, arrival, cabin class, fare. Visible once `flightOffers.isPresent()`.
    - Phase panel 2 (Seat selection): selected flight summary, seat code, cabin class, total cost. Visible once `seatSelection.isPresent()`.
    - Confirmation gate panel (only visible when `status = AWAITING_CONFIRMATION`): a yellow **Confirm booking** button with the itinerary summary above it. Clicking calls `POST /api/bookings/{id}/confirm`.
    - Phase panel 3 (Booking confirmation): confirmation code, passenger reference, full itinerary. Visible once `confirmation.isPresent()`.
    - Rejection-log strip (only visible if the booking has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: CREATED=muted, SEARCHING=blue, SEARCH_DONE=blue, SELECTING=yellow, SEAT_SELECTED=yellow, AWAITING_CONFIRMATION=orange, CONFIRMED=teal, BOOKING=blue, COMMITTED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A booking in `SELECTING` shows phase panel 1 only (panel 2 is not yet present). A booking in `AWAITING_CONFIRMATION` shows panels 1 and 2 plus the confirmation gate. This is the visual proof that the typed handoff between phases is the only path information travels — and that the booking gate is a genuine pause, not a cosmetic delay.
