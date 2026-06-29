# UI mockup — story-teller

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: StoryTeller</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Story<span class="accent">Teller</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a genre (Fantasy / Mystery / Sci-Fi / Romance / Horror / Custom) and a seeded prompt, or type your own.
  3. Set optional style constraints (tone, word count, point of view) and click **Generate story**.
  4. Watch the card transition through ENRICHED → GENERATING → STORY_RECORDED → SCORED.
- Card **How it works**: one paragraph on submit → enrich → generate → score; one paragraph on the two governance mechanisms (guardrail, on-decision eval) and the content-safety filter.
- Card **Components**: rows per component (StoryEntity, PromptEnricher, StoryWorkflow, StoryTellerAgent, StoryGuardrail, QualityScorer, StoryView, StoryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions_surface: full-automation` declaration is filled and prominent. `oversight.human_in_loop = false` and `content_safety_filter_before_llm: true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a prompt. <span class="accent">Read the story.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: `Genre` dropdown (Fantasy / Mystery / Sci-Fi / Romance / Horror / Custom), `Prompt` textarea (with a "Load seeded example" link that fills the prompt), `Tone` select (WHIMSICAL / GRITTY / LITERARY / NEUTRAL), `Word count` select (100 / 300 / 600), `Point of view` select (FIRST / THIRD), `Requested by` text input, and a yellow `Generate story` button.
    - Live list below: one card per story, newest-first. Each card shows status pill, quality score chip (when scored), genre badge, prompt excerpt, age.
  - **Right column** — Selected-story detail.
    - Header: status pill + quality score chip + genre badge + story title.
    - Original prompt: the raw prompt text submitted.
    - Enriched prompt preview: a monospace block of the safe prompt, with content tag chips above (e.g., `mystery`, `crime`, `locked-room`). For BLOCKED stories, shows the block reason chip instead.
    - Story body: the narrative text with paragraph breaks preserved.
    - Author's note: italic paragraph below the body.
    - Quality section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: REQUESTED=muted, ENRICHED=blue, BLOCKED=red, GENERATING=yellow, STORY_RECORDED=blue, SCORED=green, FAILED=red.
- Genre badge colours: Fantasy=purple, Mystery=teal, Sci-Fi=cyan, Romance=pink, Horror=orange, Custom=muted.
- Quality score chip: 1–2=red, 3=yellow, 4–5=green.

The raw prompt is always displayed — there is no PII concern in this domain. The enriched prompt's content tags give the user visibility into what the safety filter found.
