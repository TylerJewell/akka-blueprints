# UI mockup — akka-basic

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Single-Prompt Workflow</title>`.

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
- Headline: `Single-Prompt <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded prompt (summarisation / extraction / classification) or type your own.
  3. Click **Run prompt**.
  4. Watch the card transition through PROCESSING → COMPLETED.
- Card **How it works**: one paragraph on submit → workflow → agent → result; one paragraph on the after-agent-response guardrail and what it checks.
- Card **Components**: rows per component (PromptEntity, PromptWorkflow, PromptAgent, OutputGuardrail, PromptView, PromptEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and displayed in muted italic. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are filled and prominent. `operations.agent_count = 1` and `agent_pattern = single-agent` are filled.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a prompt. <span class="accent">Read the result.</span>`. Subtitle: `One agent, one guardrail around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Prompt category` (Summarise / Extract / Classify / Auto), `Prompt text` textarea (with a "Load seeded example" link that fills the textarea), `Submitted by` text input, and a yellow `Run prompt` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, category badge (when result landed), confidence bar (when result landed), age.
  - **Right column** — Selected-run detail.
    - Header: status pill + category badge + `runId`.
    - Submitted prompt: the original `promptText` in a monospace block with the `categoryHint` chip above it.
    - Result section: category badge, confidence score bar (0–100 %), and the `outputText` in a readable paragraph.
    - No result yet: a spinner if status is `PROCESSING`, a red failure message if status is `FAILED`.
- Status pill colours: SUBMITTED=muted, PROCESSING=yellow, COMPLETED=green, FAILED=red.
- Category badge colours: SUMMARISE=blue, EXTRACT=purple, CLASSIFY=teal.
- Confidence bar: green above 0.75, yellow 0.50–0.75, red below 0.50.
