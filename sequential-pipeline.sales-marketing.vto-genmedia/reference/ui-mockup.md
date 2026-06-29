# UI mockup — vto-genmedia

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: GenMedia for Commerce</title>`.

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
- Headline: `GenMedia for <span class="accent">Commerce</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a garment from the seeded catalogue, choose a model preset, and optionally check **Include video clip**.
  3. Click **Run try-on** and watch the card transition through PREPARING → ASSETS_READY → GENERATING → MEDIA_GENERATED → VALIDATING → VALIDATED → EVALUATED.
  4. Inspect the safety-rejection log strip on the card if the guardrail intercepted any output during generation.
- Card **How it works**: one paragraph on the three task phases (PREPARE → GENERATE → VALIDATE) and the typed handoff between them; one paragraph on the two governance mechanisms (image safety guardrail, render quality eval).
- Card **Components**: rows per component (TryOnEntity, VtoPipelineWorkflow, VtoAgent, PrepareTools, GenerateTools, ValidateTools, ImageSafetyGuardrail, RenderQualityScorer, TryOnView, TryOnEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration is notable — the garment catalogue ID and pose preset are the only inputs; no shopper identity is stored. `decisions.authority_level = recommend-only` (the shopper decides independently after viewing the composite) and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` appear faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Pick a garment. <span class="accent">See yourself in it.</span>`. Subtitle: `One agent, three task phases, one safety gate between generation and the data layer.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a **Garment** dropdown (seeded garments), a **Model preset** dropdown (`standing-front`, `standing-side`, `seated`), an **Include video clip** checkbox, and a yellow `Run try-on` button.
    - Live list below: one card per try-on request, newest-first. Each card shows status pill, quality score chip (when eval landed), garment name, age, and a small red dot if any safety rejection fired during this request.
  - **Right column** — Selected-request detail.
    - Header: status pill + quality score chip + garment display name.
    - Phase panel 1 (Asset summary): a two-column table showing garment metadata (name, category, primary colour swatch) and model preset (pose, background colour, target dimensions). Visible once `assets` is non-null.
    - Phase panel 2 (Generated media): composite image thumbnail (click to enlarge) and, if present, a video clip player placeholder with the clip URL. `MediaResult.generationParams` shown as a small key–value list. Visible once `media` is non-null.
    - Phase panel 3 (Validation): two badge rows — Dimension (green tick or red cross + actual/expected ratio) and Colour fidelity (green tick or red cross + deltaE value + colour swatches). `safetyVerdict` badge (`CLEARED` in green, `FLAGGED` in amber). Visible once `validatedMedia` is non-null.
    - Quality section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Safety-rejection log strip (only visible if the request has any `safetyRejections`): a small table with category, reason, and time.
- Status pill colours: CREATED=muted, PREPARING=blue, ASSETS_READY=blue, GENERATING=yellow, MEDIA_GENERATED=yellow, VALIDATING=blue, VALIDATED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A request in `GENERATING` shows panel 1 (asset summary) and a spinner in the panel 2 slot. This is the visual proof that the typed handoff between phases is the only path information travels.
