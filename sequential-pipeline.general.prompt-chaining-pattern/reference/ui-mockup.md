# UI mockup — prompt-chaining-workflow

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Prompt Chaining Workflow</title>`.

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
- Headline: `Prompt Chaining <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded prompts (or type your own) and click **Start drafting**.
  3. Watch the card transition through OUTLINING → OUTLINED → DRAFTING → DRAFTED → REFINING → REFINED → EVALUATED.
  4. Inspect the rejection-log strip on the card if the response guardrail caught a structurally incomplete document.
- Card **How it works**: one paragraph on the three chained steps (OUTLINE → DRAFT → REFINE) and the typed handoff between them; one paragraph on the two governance mechanisms (response guardrail at the output boundary, completeness eval after commitment).
- Card **Components**: rows per component (DraftEntity, DraftingWorkflow, DraftingAgent, OutlineTools, DraftTools, RefineTools, OutputGuardrail, QualityScorer, DraftView, DraftEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (the prompt is the only user input; templates and citations come from an in-process corpus). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Type a prompt. <span class="accent">Read the document.</span>`. Subtitle: `One agent, three chained steps, one output gate before the result lands.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Prompt` (with a "Pick a seeded prompt" dropdown that fills it), and a yellow `Start drafting` button.
    - Live list below: one card per draft, newest-first. Each card shows status pill, quality score chip (when eval landed), prompt text (truncated), age, and a small red dot if any guardrail rejection fired during this draft.
  - **Right column** — Selected-draft detail.
    - Header: status pill + quality score chip + prompt text.
    - Step panel 1 (Outline): a section list with heading and description columns. Visible once `outline` is present.
    - Step panel 2 (Draft): per-section body paragraphs and a citations table (label, url, snippet). Visible once `draft` is present.
    - Step panel 3 (Refined document): title, abstract paragraph, then a per-section block (heading, body, citation markers as chips). Visible once `refinedDocument` is present.
    - Quality section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if `guardrailRejections` is non-empty): a small table with check name, reason, and time.
- Status pill colours: CREATED=muted, OUTLINING=blue, OUTLINED=blue, DRAFTING=yellow, DRAFTED=yellow, REFINING=blue, REFINED=blue, EVALUATED=green, FAILED=red.

Each step panel renders only when its data is present on the row record. A draft in `REFINING` shows step panels 1 and 2 (step panel 3 with a "in progress" spinner if the agent has not yet returned). This is the visual proof that the typed handoff between steps is the only path information travels through the chain.
