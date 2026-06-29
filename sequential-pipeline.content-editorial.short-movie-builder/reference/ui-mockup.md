# UI mockup — short-movie-agents

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Short Movie Builder</title>`.

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
- Headline: `Short Movie <span class="accent">Builder</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded creative briefs (or type your own) and click **Produce movie**.
  3. Watch the card transition through SCRIPTING → SCRIPTED → STORYBOARDING → STORYBOARDED → ASSEMBLING → ASSEMBLED → REVIEWING → REVIEWED.
  4. Inspect the safety-block log strip on the card if any content-safety rejections fired.
- Card **How it works**: one paragraph on the four task phases (SCRIPT → STORYBOARD → ASSEMBLE → REVIEW) and the typed handoff between them; one paragraph on the content-safety guardrail and the on-decision coherence scorer.
- Card **Components**: rows per component (MovieEntity, MovieProductionWorkflow, MovieAgent, ScriptTools, StoryboardTools, AssemblyTools, ReviewTools, ContentSafetyGuard, PackageScorer, MovieView, MovieEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 4 function-tool classes, 1 Guardrail, 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the creative brief is the only user input; scenes come from an in-process corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (H1). ID badge coloured orange (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Type a brief. <span class="accent">Get a movie package.</span>`. Subtitle: `One agent, four production phases, one safety gate on every result.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Creative brief` (with a "Pick a seeded brief" dropdown that fills it), and a yellow `Produce movie` button.
    - Live list below: one card per production, newest-first. Each card shows status pill, coherence score chip (when scoring has completed), brief excerpt, age, and a small orange dot if any content-safety block fired during this production.
  - **Right column** — Selected-production detail.
    - Header: status pill + coherence score chip + brief excerpt.
    - Phase panel 1 (Script): a table with columns sceneId, type, description, dialogueLine. Visible once `script.isPresent()`.
    - Phase panel 2 (Storyboard): a shot list with columns shotId, sceneId, framing, duration. Visible once `storyboard.isPresent()`.
    - Phase panel 3 (Assembled Package): a package-scene table with columns orderIndex, sceneId, shotId, assetPlaceholder, and a total-runtime badge. Visible once `assembledPackage.isPresent()`.
    - Phase panel 4 (Review): a coherence-check table with columns sceneId, coherent (tick/cross), note; then the review summary paragraph. Visible once `reviewResult.isPresent()`.
    - Coherence section at bottom: a 1–5 star widget and the one-line rationale from `coherenceResult`. Score ≤ 2 highlights the card border red.
    - Safety-block log strip (only visible if the production has any `safetyBlocks`): a small table with phase, violationType, reason, time.
- Status pill colours: CREATED=muted, SCRIPTING=blue, SCRIPTED=blue, STORYBOARDING=yellow, STORYBOARDED=yellow, ASSEMBLING=blue, ASSEMBLED=blue, REVIEWING=yellow, REVIEWED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A production in `ASSEMBLING` shows panels 1, 2, and 3 (panel 3 with a spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
