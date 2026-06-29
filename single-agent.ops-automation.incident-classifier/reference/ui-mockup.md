# UI mockup — incident-classifier

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: ITSM Incident Categorization Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See AKKA-EXEMPLAR-LESSONS.md Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `ITSM Incident <span class="accent">Categorization Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded incident (database connection failure / VPN authentication outage / file-share permission denial) and click **Load seeded example**, or paste your own short and long descriptions.
  3. Click **Submit incident**.
  4. Watch the card transition through TAXONOMY_VALIDATED → CLASSIFYING → CLASSIFICATION_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → validate → classify → eval; one paragraph on the two governance mechanisms (on-decision taxonomy validator, continuous-accuracy rolling window).
- Card **Components**: rows per component (IncidentEntity, VocabularyValidator, ClassificationWorkflow, IncidentClassifierAgent, AccuracyEvaluator, ClassificationView, ClassificationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 AccuracyEvaluator + 1 TaxonomyTable as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `decisions.authority_level = automated-with-override` declaration is prominent. `oversight.human_on_loop = true` and `failure.failure_modes` (wrong-category, accuracy-drift, misrouted-high-priority-incident) are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Two sections: a 5-column table with two rows (E1, E2), followed by the continuous-accuracy panel.
- ID badges coloured: E1 blue (eval-event), E2 teal (eval-periodic).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.
- **Accuracy panel** (below the table): fetches `/api/incidents/accuracy` on mount and renders a percentage badge (e.g. `90.5% fully correct`) plus a 7-bar sparkline of daily counts. Refreshes every 10 s via `setInterval`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an incident. <span class="accent">Get a classification.</span>`. Subtitle: `One agent, two evaluation layers around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded examples` (database failure / VPN outage / file-share denial / custom), `Short description` text input, `Long description` textarea (with **Load seeded example** button that fills both fields), `Caller ID` text input, `Priority hint` dropdown (P1–P4), and a yellow `Submit incident` button.
    - Live list below: one card per incident, newest-first. Each card shows status pill, category chip (when classified), eval score chip (when evaluated), short description truncated to 60 chars, age.
  - **Right column** — Selected-incident detail.
    - Header: status pill + category chip + subcategory chip + eval score chip + short description.
    - Submission fields: caller ID, priority hint, submitted-at timestamp, long description (collapsed to 5 lines with "show more" toggle).
    - Taxonomy scope summary: a small tile showing category count, subcategory count, CI count from `scope`.
    - Classification result: category chip, subcategory chip, affected CI label, and the agent's `rationale` sentence.
    - Eval section at bottom: a 1–5 star widget, three boolean indicators (Category in taxonomy / Subcategory valid / CI in registry), and the one-line rationale. Score ≤ 2 highlights the card border red. A "Review classification" button appears when score ≤ 2 (clicking it opens the classification detail in an editable mode — edit is out of scope for the blueprint; the button is a placeholder that shows a toast).
- Status pill colours: SUBMITTED=muted, TAXONOMY_VALIDATED=blue, CLASSIFYING=yellow, CLASSIFICATION_RECORDED=blue, EVALUATED=green, FAILED=red.
- Category chip colours: Network=indigo, Database=amber, Security=red, Storage=teal, Application=violet, Identity=cyan, unknown=grey.
- Eval score chip: 4–5=green, 3=yellow, 1–2=red.
