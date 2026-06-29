# UI mockup — skill-patterns-tutorial

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Skill Patterns Tutorial</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. Canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Skill<span class="accent">Patterns</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a pattern card (Inline / File-based / External / Meta creator).
  3. Fill in the input fields and click **Run**.
  4. Watch the run card appear in the live list and transition to COMPLETED with its output.
- Card **How it works**: one paragraph on request → agent dispatch → skill execution → result recording; one paragraph on the four patterns and what differentiates them at the wiring level.
- Card **Components**: rows per component (SkillRunEntity, SkillRunWorkflow, SkillDemoAgent, SkillToolStub, SkillRunView, SkillRunEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 3 HttpEndpoints including SkillToolStub).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Skill-wiring table: four rows (Inline / File-based / External / Meta creator) with columns: Pattern, Wiring call, Load time, Recompile to change.
- Compressed comp-row table with Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is prominent. `decisions.authority_level = informational-only` and `oversight.human_on_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Explanatory paragraph: "This baseline wires no governance controls. The sample's purpose is skill-pattern pedagogy. A deployer building on this pattern would add controls appropriate to their use case." The table body is empty; the five-column header is still rendered.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Pick a pattern. <span class="accent">See it work.</span>`. Subtitle: `One agent, four skill-wiring approaches.`
- Layout: two-column.
  - **Left column** — Pattern cards + live list.
    - Four pattern cards arranged in a 2×2 grid. Each card:
      - Header: pattern name badge + wiring-level tag (`inline` / `file-based` / `external` / `meta-creator` in a muted chip).
      - Body: one-sentence description of how the skill is wired.
      - Input panel (shown when the card is selected):
        - INLINE: `Topic` text input + `Style` selector (concise / detailed / bullet-list).
        - FILE_BASED: `Text to summarize` textarea (max 1000 characters).
        - EXTERNAL: `Lookup key` text input + a small hint showing the five seeded keys.
        - META_CREATOR: `Skill name` text input + `Skill description` textarea.
      - `Run` button (yellow). `Load example` link pre-fills the inputs from seed-inputs.jsonl.
    - Mini live list below the grid: one row per run, newest-first. Row shows status pill, pattern badge, age, and first 80 chars of output.
  - **Right column** — Selected-run detail.
    - Header: status pill + pattern badge + `runId`.
    - Pattern name + wiring note (italic).
    - Output block: monospace text area showing the full output.
    - For META_CREATOR runs with status COMPLETED: a collapsible `SkillDefinition` JSON block + a **Register** button. Clicking Register sends `POST /api/skills` with the JSON; on 201, a success chip appears on the card and the fifth pattern card appears in the grid (if not already present).
- Status pill colours: REQUESTED=muted, RUNNING=yellow, COMPLETED=green, FAILED=red.
- Pattern badge colours: INLINE=blue, FILE_BASED=teal, EXTERNAL=orange, META_CREATOR=purple.
