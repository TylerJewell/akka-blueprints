# UI mockup — deterministic-multi-stage-agent-pipeline

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Deterministic Multi-Stage Agent Pipeline</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents (clicking a tab and seeing a blank because a hidden zombie panel occupied the index).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Deterministic Multi-Stage <span class="accent">Agent Pipeline</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded prompts (or type your own) and click **Run pipeline**.
  3. Watch the card transition through OUTLINING → OUTLINED → BODY_WRITING → BODY_WRITTEN → ENDING_WRITING → ENDING_WRITTEN → VALIDATED.
  4. Inspect the rejection-log strip on the card if any stage-output rejections fired.
- Card **How it works**: one paragraph on the three task stages (OUTLINE → WRITE_BODY → WRITE_ENDING) and the typed handoff between them; one paragraph on the two governance mechanisms (stage-output guardrail, structural coherence eval).
- Card **Components**: rows per component (StoryEntity, StoryPipelineWorkflow, StoryAgent, OutlineTools, BodyTools, EndingTools, StageOutputGuardrail, StoryStructureValidator, StoryView, StoryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Validator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the prompt is the only user input; genre data comes from an in-process corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Type a prompt. <span class="accent">Read the story.</span>`. Subtitle: `One agent, three task stages, one structural gate after each.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Prompt` (with a "Pick a seeded prompt" dropdown that fills it), and a yellow `Run pipeline` button.
    - Live list below: one card per story, newest-first. Each card shows status pill, validation score chip (when validation landed), prompt, age, and a small red dot if any guardrail rejection fired during this story.
  - **Right column** — Selected-story detail.
    - Header: status pill + validation score chip + prompt.
    - Stage panel 1 (Outline): a list of beats with beat id, position, and text. Also shows the detected genre chip. Visible once `outline.isPresent()`.
    - Stage panel 2 (Body): a list of paragraph cards, each showing the referenced beat id and the paragraph text. Visible once `body.isPresent()`.
    - Stage panel 3 (Ending): the closing paragraph text block, then an arc list (arc label + referenced paragraph ids). Visible once `ending.isPresent()`.
    - Validation section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the story has any `guardrailRejections`): a small table with stage, field, reason, time.
- Status pill colours: CREATED=muted, OUTLINING=blue, OUTLINED=blue, BODY_WRITING=yellow, BODY_WRITTEN=yellow, ENDING_WRITING=blue, ENDING_WRITTEN=blue, VALIDATED=green, FAILED=red.

Each stage panel renders only when its data is present on the row record. A story in `BODY_WRITING` shows panels 1 and 2 (panel 2 with an "in progress" spinner if the agent has not yet returned the Body). This is the visual proof that the typed handoff between stages is the only path information travels.
