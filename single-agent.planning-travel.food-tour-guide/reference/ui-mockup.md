# UI mockup — gemma-food-tour-guide

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Gemma Food Tour Guide</title>`.

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
- Headline: `Gemma Food <span class="accent">Tour Guide</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a city (Tokyo / Barcelona / Mexico City), set trip duration, and select dietary styles — or load a seeded example with one click.
  3. Click **Plan my tour**.
  4. Watch the card transition through PREFERENCES_VALIDATED → GENERATING → ITINERARY_RECORDED → SCORED.
- Card **How it works**: one paragraph on submit → validate → generate → score; one paragraph on the two governance mechanisms (guardrail, on-decision eval).
- Card **Components**: rows per component (TourRequestEntity, PreferenceValidator, TourWorkflow, FoodTourAgent, ItineraryGuardrail, CoverageScorer, TourView, TourEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block:

```js
mermaid.initialize({
  theme: 'dark',
  themeVariables: {
    nodeTextColor: '#e6edf3',
    stateLabelColor: '#e6edf3',
    transitionLabelColor: '#cccccc'
  }
});
```

```css
.mermaid svg .stateLabel { fill: #e6edf3 !important; }
.mermaid svg .edgeLabel foreignObject { overflow: visible; }
.mermaid svg .edgeLabel { color: #cccccc; }
```

Without these, state labels render black-on-black and edge labels clip at the top.

- Component table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `personal_health_notes_normalized_before_llm: true` declaration is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Plan a food tour. <span class="accent">Read the itinerary.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Request panel + live list.
    - Request panel: dropdown `City` (Tokyo / Barcelona / Mexico City), number input `Duration (days)` (1–7), checkbox group `Dietary styles` (vegetarian / vegan / pescatarian / omnivore / gluten-free / halal / kosher), radio group `Budget tier` (Street food / Mid-range / Fine dining), textarea `Notes (optional)`, text input `Requested by`, and a yellow `Plan my tour` button. A "Load seeded example" link pre-fills all fields.
    - Live list below: one card per tour request, newest-first. Each card shows status pill, decision badge (when itinerary landed), coverage score chip (when coverage landed), city + duration, age.
  - **Right column** — Selected-request detail.
    - Header: status pill + decision badge + coverage score chip + city + duration.
    - Validated preferences: a tag cloud of normalized category labels and budget tier chip.
    - Day-by-day itinerary: one collapsible section per day. Each section lists venue cards. Each venue card shows meal slot badge, venue name, neighborhood, dietary category chip, cultural note (italic), and description.
    - Coverage section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: REQUESTED=muted, PREFERENCES_VALIDATED=blue, GENERATING=yellow, ITINERARY_RECORDED=blue, SCORED=green, FAILED=red.
- Decision badge colours: FULL_PLAN=green, PARTIAL_PLAN=yellow, NEEDS_MORE_INFO=red.
- Meal slot badge colours: BREAKFAST=amber, LUNCH=green, DINNER=blue, SNACK=muted, MARKET=purple.
- Budget tier chip: STREET_FOOD=muted, MID_RANGE=blue, FINE_DINING=yellow.
