# UI mockup — personalized-shopper

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Personalized Shopper</title>`.

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
- Headline: `Personalized <span class="accent">Shopper</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded preference profile (electronics / apparel / home goods) or fill in the form fields directly.
  3. Click **Get recommendations**.
  4. Watch the session card transition through PROFILE_SANITIZED → RECOMMENDING → RECOMMENDATIONS_READY → FRESHNESS_SCORED.
- Card **How it works**: one paragraph on submit → sanitize → recommend → freshness; one paragraph on the PII sanitizer governance mechanism and why personal identifiers are stripped before the model call.
- Card **Components**: rows per component (ShoppingSessionEntity, ProfileSanitizer, RecommendationWorkflow, ShoppingAdvisorAgent, FreshnessScorer, RecommendationView, ShoppingEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Scorer as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (S1). ID badge coloured green (sanitizer).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit preferences. <span class="accent">Get recommendations.</span>`. Subtitle: `One agent, one governance mechanism around it.`
- Layout: two-column.
  - **Left column** — Preference form + live list.
    - Preference form: dropdown `Seeded profile` (electronics buyer / apparel shopper / home-goods renovator / custom), `Preferred categories` multi-select, `Preferred brands` text input (comma-separated), `Price range` dual-input ($min – $max), `Excluded categories` multi-select, `Free-text note` textarea, `Catalog snapshot` dropdown (electronics / apparel / home-goods / mixed), and a yellow `Get recommendations` button.
    - Live list below: one card per session, newest-first. Each card shows status pill, outcome badge (when recommendations land), freshness score chip (when freshness lands), and age.
  - **Right column** — Selected-session detail.
    - Header: status pill + outcome badge + freshness score chip.
    - Preference summary: category pills, brand pills, price range, excluded categories, free-text note (PII stripped).
    - PII stripped section: a small chip list of categories that were stripped (`email`, `phone`, `loyalty-card`, `person-name`).
    - Recommendation list: ranked cards, each showing rank badge, productName, category, price, confidence chip, and rationale paragraph.
    - Freshness section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: STARTED=muted, PROFILE_SANITIZED=blue, RECOMMENDING=yellow, RECOMMENDATIONS_READY=blue, FRESHNESS_SCORED=green, FAILED=red.
- Outcome badge colours: MATCHED=green, PARTIAL_MATCH=yellow, NO_MATCH=muted.
- Confidence chip colours: HIGH=green, MEDIUM=blue, LOW=muted.

The raw PII fields (shopperEmail, shopperPhone, loyaltyCardNumber, shopperName) are never displayed on this screen. A developer needing them for audit fetches `GET /api/sessions/{id}` and reads `profile.*` from the JSON. This is intentional: the UI demonstrates that the model's input contains no personal identifiers, while the audit trail preserves them on the entity.
