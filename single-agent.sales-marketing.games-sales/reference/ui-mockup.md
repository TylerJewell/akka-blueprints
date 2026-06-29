# UI mockup — games-sales

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Video Games Sales Assistant</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Video Games <span class="accent">Sales Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded query (PS5 action / Switch RPG / Xbox puzzle) or type your own.
  3. Click **Get recommendations**.
  4. Watch the session card transition through QUERYING → RECOMMENDED; read the ranked suggestions.
- Card **How it works**: one paragraph on query → agent call → guardrail → recommendation recorded; one paragraph on the guardrail mechanism (catalog validation, confidence score range, non-empty suggestions).
- Card **Components**: rows per component (SessionEntity, SessionWorkflow, SalesAssistantAgent, RecommendationGuardrail, RecommendationView, SalesEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = false` are the distinctive answers for a customer-facing automated flow. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask for a game. <span class="accent">Get a recommendation.</span>`. Subtitle: `One agent, one guardrail, real-time session state.`
- Layout: two-column.
  - **Left column** — Query panel + live list.
    - Query panel: dropdown `Load seeded query` (PS5 co-op action / Switch RPG / Xbox puzzle / custom), `Platform` text input, `Genre` text input, `Max budget ($)` number input, `Customer ID` text input, a large **What are you looking for?** textarea, and a styled **Get recommendations** button.
    - Live list below the panel: one card per session, newest-first. Each card shows status pill, age, customer ID, and a one-line snippet of the query.
  - **Right column** — Selected-session detail.
    - Header: status pill + session ID + customer ID.
    - Preferences summary: query text, platform chip, genre chip, budget chip.
    - Recommendation section (when status ≥ RECOMMENDED): a ranked list of suggestion cards. Each suggestion card shows:
      - Title + platform label.
      - Price formatted as `$XX.XX`.
      - Confidence score bar (0–100% width, colour: green ≥ 0.8, yellow 0.5–0.79, red < 0.5).
      - Rationale sentence in italic.
      - Upsell chip (amber, shown only when `upsellNote` is non-null).
    - Summary paragraph below the list.
    - Follow-up input: a textarea labeled **Refine your request** and a **Send follow-up** button. Visible only when status is `RECOMMENDED` or `FOLLOW_UP_RECOMMENDED`.
    - Follow-up recommendation section (when status = `FOLLOW_UP_RECOMMENDED`): same card layout as above, with a `Follow-up` header label.
- Status pill colours: QUERYING=yellow, RECOMMENDED=green, FOLLOW_UP_QUERYING=yellow, FOLLOW_UP_RECOMMENDED=green, FAILED=red.
- Confidence bar colours: ≥ 0.8 = green, 0.5–0.79 = amber, < 0.5 = red.
