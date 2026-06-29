# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build (Lesson 17). Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid — matching `governance.html`. Browser title: `<title>Akka Sample: Hello Agent</title>`. Content wraps in `.wrap{ max-width:1080px }` with no horizontal scroll (Lesson 12).

## Tabs

1. **Overview** — eyebrow ("Overview") + headline ("Single-agent question answerer") with no subtitle, then four cards: **Try it** (just `/akka:build`, no env-var block), **How it works** (one agent, one ANSWER task, one output check), **Components** (the inventory), **API contract** (the top-level paths).
2. **Architecture** — the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, entity model), each in a `.diagram-card`. Carries the Lesson 24 CSS overrides: white `.stateLabel` / `.nodeLabel`, `overflow:visible` on edge-label `foreignObject`, and `transitionLabelColor:#cccccc` in the mermaid theme variables.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style. Each answer is a row: question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic ("To be completed by deployer").
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same style, one `matrix-row` for the single control `G1`. The label column carries a colored mechanism pill (`guardrail` red). The value column shows the name, rationale, and implementation notes.
5. **App UI** — the live sample (below).

## App UI tab

- **Ask box** — a text input plus an **Ask** button. Submitting `POST /api/ask` with `{ question }`, then clears the box.
- **Questions list** — live via `EventSource` on `/api/questions/sse`, newest first, upserted by `id`. Each row shows:
  - the question text and a status pill (`ASKED` muted, `ANSWERED` green, `BLOCKED` red);
  - when `ANSWERED`: the answer text and the confidence as a percentage;
  - when `BLOCKED`: the block reason, no answer text;
  - when `ASKED`: a "thinking…" indicator.

## Tab switching — MUST be attribute-based (Lesson 26)

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

Removed tabs are deleted from the DOM, never hidden with `display:none` — a hidden zombie panel shifts index-based switching and blanks the App UI tab.
