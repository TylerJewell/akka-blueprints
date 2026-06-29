# UI mockup — fun-facts

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: FunFacts</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Fun<span class="accent">Facts</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Type a topic or click a seeded example (Octopus Intelligence / Black Holes / Ancient Roman Engineering).
  3. Click **Generate facts**.
  4. Watch the card transition through PENDING → GENERATING → GENERATED.
- Card **How it works**: one paragraph on submit → generate → project; one paragraph on the single-agent pattern and why this baseline deliberately carries no governance controls.
- Card **Components**: rows per component (FactRequestEntity, FactRequestWorkflow, FactGeneratorAgent, FactRequestView, FactEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled and prominent. `decisions.authority_level = informational-only` and `oversight.human_in_loop = false` are the distinctive answers — this is a low-stakes baseline. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Display an empty-state message: "No controls are wired in this baseline. See `eval-matrix.yaml` for guidance on adding controls for production deployments."
- No table rows; no ID badges.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a topic. <span class="accent">Read the facts.</span>`. Subtitle: `One agent, no ceremony.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: three seeded-topic quick-pick buttons (Octopus Intelligence / Black Holes / Ancient Roman Engineering), a `Topic` text input (pre-fills when a quick-pick is clicked), a `Requested by` text input, and a yellow `Generate facts` button.
    - Live list below: one card per request, newest-first. Each card shows status pill, topic text, age.
  - **Right column** — Selected-request detail.
    - Header: status pill + topic text.
    - Headline fact: rendered in a prominent block with larger font.
    - Supporting facts list: one card per `Fact` showing the `area` chip, `statement` text, and `confidence` chip (HIGH=green, MEDIUM=blue, LOW=muted).
    - Empty state when no request is selected: "Select a request from the list or submit a new topic."
- Status pill colours: PENDING=muted, GENERATING=yellow, GENERATED=green, FAILED=red.
- Confidence chip colours: HIGH=green, MEDIUM=blue, LOW=muted.

The selected-request detail panel clears and reloads via SSE when the active request transitions to a new status — no polling.
