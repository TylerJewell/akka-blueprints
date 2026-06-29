# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build (Lesson 17). Inline CSS + JS; runtime CDN imports for markdown and YAML are allowed. Browser title `<title>Akka Sample: Multi-Perspective Debate</title>`. Visual style anchor: dark background, yellow accent, Instrument Sans, dot-grid (the `governance.html` style). Content fits the 1080px column with no horizontal scroll (Lesson 12).

Five left-nav tabs, in order: Overview, Architecture, Risk Survey, Eval Matrix, App UI.

## Tab 1 — Overview

Eyebrow "Overview" + headline naming the sample type (multi-perspective debate), **no subtitle**, then four cards:
- **Try it** — the single command `/akka:build`. No env-var export block.
- **How it works** — Advocate and Critic alternate up to five rounds; the Synthesizer concludes; the synthesis is guardrailed and scored.
- **Components** — the inventory from README "What you'll get".
- **API contract** — the top-level endpoint surface.

## Tab 2 — Architecture

Renders the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER). Mermaid initialised with the Akka theme variables **and** the Lesson 24 CSS overrides: white `color`/`fill` on every state-label DOM path, `overflow:visible` on every edge-label `foreignObject`, and `transitionLabelColor:#cccccc`.

## Tab 3 — Risk Survey

Fetches `/api/metadata/risk-survey` and renders it in the `matrix-card` / `matrix-row` style: each answer is a row with the question label on the left (yellow) and the value on the right. Any value matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` renders in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

Fetches `/api/metadata/eval-matrix` and renders it in the same `matrix-card` / `matrix-row` style, one row per control (G1, E1). The label column carries a colored mechanism pill: `guardrail` red, `eval-event` blue. The value column shows the control name, rationale, and implementation note.

## Tab 5 — App UI

- A topic text input and a **Start debate** button → `POST /api/debates`.
- A live list of debates via `GET /api/debates/sse`, newest first. Each row shows the topic, a status chip (`PENDING`/`DEBATING`/`SYNTHESIZING`/`CONCLUDED`/`FAILED`), and `currentRound / maxRounds`.
- Clicking a row expands it to show every recorded round (advocate argument and critic rebuttal side by side), then, once `CONCLUDED`, the conclusion, the key-arguments list, and the quality score. A `FAILED` debate shows its `failedReason`.

## Tab switching MUST be attribute-based (Lesson 26)

Switch by `data-tab` / `data-panel` attribute, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

There are exactly five `.nav-tab` elements (`data-tab="0".."4"`) and five `.tab-panel` elements (`data-panel="0".."4"`). Any removed tab's panel is **deleted** from the DOM, never hidden with `display:none` — a hidden zombie panel breaks index-based switching and leaves the App UI blank.
