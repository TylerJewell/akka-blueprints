# UI mockup — maps-trip-planner

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Maps Trip Planner</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Maps Trip <span class="accent">Planner</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Fill in origin, destination(s), dates, party size, and preferences — or load a seeded example (Paris / Amalfi / Tokyo).
  3. Click **Plan my trip**.
  4. Watch the card transition through SCREENED → PLANNING → ITINERARY_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → screen → plan → eval; one paragraph on the three governance mechanisms (destination screener, guardrail, on-decision eval).
- Card **Components**: rows per component (TripRequestEntity, RequestScreener, PlanningWorkflow, TripPlannerAgent, ItineraryGuardrail, CoverageScorer, MockMapsClient, ItineraryView, PlanningEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer + 1 Maps stub as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `location-fine-grained: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. `maps-api-terms-of-service-reviewed` is a deployer-specific field faded in muted italic as `TO_BE_COMPLETED_BY_DEPLOYER`. Many other fields are similarly faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, E1). ID badges coloured: S1 orange (screener), G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a request. <span class="accent">Get an itinerary.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Request panel + live list.
    - Request panel: `Origin city` text input, `Destinations` tag input (comma-separated), `Start date` / `End date` date pickers, `Party size` number input, `Preferences` textarea (placeholder: "vegetarian meals, avoid museums, budget mid-range"), `Requested by` text input, a "Load seeded example" dropdown (Paris / Amalfi Coast / Tokyo), and a teal `Plan my trip` button.
    - Live list below: one card per trip, newest-first. Each card shows status pill, quality badge (when itinerary landed), eval score chip (when eval landed), destination(s), age.
  - **Right column** — Selected-trip detail.
    - Header: status pill + quality badge + eval score chip + destinations string.
    - Screen result: a small banner (PASS green / BLOCKED red) with the reason string.
    - Itinerary: a vertical timeline. Each day is a collapsible panel showing the `dayNarrative`, a stop list (place name, visit duration chip, short description), and a legs list (from → to, mode icon, duration).
    - Preferences coverage: a small list of preference keywords; addressed ones are ticked, unaddressed ones are muted.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, SCREENED=blue, BLOCKED=red, PLANNING=yellow, ITINERARY_RECORDED=blue, EVALUATED=green, FAILED=red.
- Quality badge colours: EXCELLENT=green, GOOD=blue, PARTIAL=yellow.
- Eval score chip: score ≤ 2 = red, 3 = yellow, 4–5 = green.
