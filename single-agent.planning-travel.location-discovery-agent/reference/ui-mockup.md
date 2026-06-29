# UI mockup — location-discovery-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Foursquare Location Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Foursquare Location <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded city area (Downtown SF / Midtown NYC / Central London) or enter your own coordinates.
  3. Type a search query and click **Find places**.
  4. Watch the card transition through COORDINATE_SANITIZED → DISCOVERING → RECOMMENDATION_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → sanitize → discover → eval; one paragraph on the three governance mechanisms (coordinate sanitizer, guardrail, on-decision eval).
- Card **Components**: rows per component (SearchEntity, CoordinateSanitizer, DiscoveryWorkflow, LocationDiscoveryAgent, RecommendationGuardrail, EvaluationScorer, SearchView, SearchEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `location-fine-grained: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, E1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Search a location. <span class="accent">Discover places.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Search panel + live list.
    - Search panel: three seeded-area buttons (Downtown SF / Midtown NYC / Central London) that fill the lat/lng fields; `Latitude` and `Longitude` numeric inputs; `Radius` selector (500 m / 1 km / 2 km); `Category` filter dropdown (All / Food & Drink / Retail / Arts & Entertainment / Outdoors); `Search query` textarea; `Submitted by` text input; and a yellow `Find places` button.
    - Live list below: one card per search, newest-first. Each card shows status pill, query snippet (first 60 chars), age, and — when landed — the top place name with its relevance score chip and the eval score chip.
  - **Right column** — Selected-search detail.
    - Header: status pill + query text + bounding-box token chip.
    - Candidate summary: candidate place count, radius, category filter.
    - Recommendation list: ranked place cards, each showing name, category, distance badge, relevance score bar (1–10), and the one-sentence rationale.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, COORDINATE_SANITIZED=blue, DISCOVERING=yellow, RECOMMENDATION_RECORDED=blue, EVALUATED=green, FAILED=red.
- Relevance score bar: gradient green (high) → yellow (mid) → red (low).

The raw coordinates are never displayed on this screen — only the bounding-box token. Users who need the raw coordinates fetch `/api/searches/{id}` and read `request.rawLatitude` / `request.rawLongitude` from the JSON. This demonstrates that the model's input is the coarse token, even though the audit trail keeps the precise position.
