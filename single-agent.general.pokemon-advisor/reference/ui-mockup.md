# UI mockup — pokemon-advisor

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Pokemon Team Advisor</title>`.

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
- Headline: `Pokemon Team <span class="accent">Advisor</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a battle format (VGC / Singles / Double-Battle / Casual) and load a seeded roster, or type in your own species names.
  3. Click **Get advice**.
  4. Watch the card transition through VALIDATED → ADVISING → RECOMMENDATION_RECORDED → SCORED.
- Card **How it works**: one paragraph on submit → validate → advise → score; one paragraph on the two governance mechanisms (before-agent-response guardrail, on-decision coverage eval).
- Card **Components**: rows per component (RosterEntity, RosterValidator, AdvisoryWorkflow, TeamAdvisorAgent, RecommendationGuardrail, CoverageScorer, AdvisoryView, AdvisoryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is prominent — this is a toy domain with no personal data beyond a trainer name. `decisions.authority_level = inform-only` and `oversight.human_in_loop = true` are the distinctive answers. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a roster. <span class="accent">Read the advice.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Battle format` (VGC / Singles / Double-Battle / Casual), `Trainer name` text input, up to 6 species inputs (Slot 1–6) with optional nickname field per slot, a "Load seeded example" link that fills the format and all species fields, and a yellow `Get advice` button.
    - Live list below: one card per advisory, newest-first. Each card shows status pill, verdict badge (when recommendation landed), coverage score chip (when score landed), trainer name + format, age.
  - **Right column** — Selected-advisory detail.
    - Header: status pill + verdict badge + coverage score chip + trainer name + format.
    - Submitted roster: a compact table with slot number, species, nickname, and any validation warnings per slot.
    - Recommendation summary: the agent's 1–3-sentence paragraph.
    - Slot recommendations table: columns slot, species, role (coloured chip), suggested moves (comma-separated), rationale.
    - Coverage gaps list: each gap shows the type, severity chip, and suggestion text.
    - Coverage score section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, VALIDATED=blue, ADVISING=yellow, RECOMMENDATION_RECORDED=blue, SCORED=green, FAILED=red.
- Verdict badge colours: STRONG=green, VIABLE=blue, NEEDS_ADJUSTMENT=yellow.
- Role chip colours: PHYSICAL_SWEEPER=orange, SPECIAL_SWEEPER=purple, WALL=teal, SUPPORT=green, LEAD=yellow, UTILITY=muted.
- Gap severity chip colours: MINOR=muted, MODERATE=blue, SIGNIFICANT=red.
