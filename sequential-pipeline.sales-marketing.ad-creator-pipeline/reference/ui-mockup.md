# UI mockup — ad-creator-pipeline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: AI Ads Generator</title>`.

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
- Headline: `AI Ads <span class="accent">Generator</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded products (or paste your own URL) and click **Generate ad**.
  3. Watch the card transition through SCRAPING → SCRAPED → COPYING → COPIED → GENERATING_VISUAL → VISUAL_GENERATED → EVALUATED.
  4. Inspect the rejection-log strip on the card if any scraping-policy or brand-safety rejections fired.
- Card **How it works**: one paragraph on the three task phases (SCRAPE → COPY → VISUAL) and the typed handoff between them; one paragraph on the two governance mechanisms (scraping-policy guardrail, brand-safety guardrail) and the quality evaluator.
- Card **Components**: rows per component (AdJobEntity, AdPipelineWorkflow, AdCreatorAgent, ScrapeTools, CopyTools, VisualTools, ScrapingPolicyGuardrail, BrandSafetyGuardrail, AdQualityScorer, AdJobView, AdJobEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 2 Guardrails, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the product URL is the only user input; product data comes from an in-process fixture corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers — a marketer must review the ad package before publication. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, G2). Both are ID badges coloured red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Paste a product URL. <span class="accent">Get an ad package.</span>`. Subtitle: `One agent, three task phases, two runtime gates.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Product URL` (with a "Pick a seeded product" dropdown that fills it with the pre-configured product URL), and a yellow `Generate ad` button.
    - Live list below: one card per ad job, newest-first. Each card shows status pill, eval score chip (when eval landed), product name, age, and coloured dots indicating whether scraping-policy rejections (orange) or brand-safety rejections (red) fired during this job.
  - **Right column** — Selected-job detail.
    - Header: status pill + eval score chip + product name.
    - Phase panel 1 (Scraped profile): a table with columns attribute, value; plus targetAudience and brandConstraints chips. Visible once `profile` is present.
    - Phase panel 2 (Ad copy): a tabbed sub-panel per format (INSTAGRAM / FACEBOOK / SEARCH) showing headline, body, and call-to-action. Visible once `copy` is present.
    - Phase panel 3 (Visual spec): image prompt in a monospace block, aspect ratio badge, style guidance line. Visible once `visual` is present.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the job has any rejections): two sub-sections — Scraping Policy and Brand Safety — each with a small table showing rule, offending text / url, and timestamp.
- Status pill colours: CREATED=muted, SCRAPING=blue, SCRAPED=blue, COPYING=yellow, COPIED=yellow, GENERATING_VISUAL=blue, VISUAL_GENERATED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A job in `COPYING` shows panels 1 and 2 (panel 2 with a "drafting…" spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
