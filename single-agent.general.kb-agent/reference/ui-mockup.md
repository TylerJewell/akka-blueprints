# UI mockup — kb-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: KbAgent</title>`.

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
- Headline: `Kb<span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded example question from the dropdown, or type your own.
  3. Click **Ask**.
  4. Watch the card transition through RETRIEVING → ANSWERING → ANSWER_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → retrieve → answer → eval; one paragraph on the groundedness evaluation mechanism.
- Card **Components**: rows per component (QueryEntity, PassageRetriever, KbDocumentConsumer, QueryWorkflow, KbAnswerAgent, GroundednessScorer, KbIndex, KbView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 Consumers, 2 HttpEndpoints, plus 1 Scorer + 1 Index as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `answer-only` decisions surface and `groundedness_score_surfaced_to_user: true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (E1). ID badge coloured blue (eval-event).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the answer.</span>`. Subtitle: `One agent, one groundedness check around it.`
- Layout: two-column.
  - **Left column** — Question submission panel + live list.
    - Submission panel: dropdown `Example questions` (product-faq-1 / product-faq-2 / policies-1 / policies-2 / release-notes-1 / custom), `Question` textarea (filled by dropdown or free-typed), `Submitted by` text input, and a yellow `Ask` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, groundedness chip (when eval landed), question excerpt (first 80 chars), age.
  - **Right column** — Selected-query detail.
    - Header: status pill + groundedness chip + question text (full).
    - Retrieved passages: a collapsible list of up to 5 passage cards, each showing document title, relevance score bar, and excerpt text.
    - Answer section: the agent's prose response.
    - Citations table: columns passage id, document title, excerpt used. Each row links back to the matching passage card above.
    - Groundedness section at bottom: a 1–5 star widget, cited-sentences / total-sentences fraction, and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, RETRIEVING=blue, ANSWERING=yellow, ANSWER_RECORDED=blue, EVALUATED=green, FAILED=red.
- Answer status badge colours: GROUNDED=green, PARTIAL=yellow, NOT_FOUND=muted.
- Groundedness chip colours: 5=green, 4=green, 3=yellow, 2=red, 1=red.
