# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build (Lesson 17). Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Browser title `<title>Akka Sample: Marketing Strategy Team</title>`. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the `governance.html` anchor).

The content column is `max-width:1080px` with no horizontal scroll (Lesson 12).

## Tab-switching MUST be attribute-based (Lesson 26)

The left-nav has five tabs. Tab switching matches by `data-tab` / `data-panel` attribute — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs are deleted from the DOM, not hidden with `display:none`. No zombie panels.

## Tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (`/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — the four mermaid diagrams (component graph, sequence, state machine, entity model) plus a component table. Mermaid carries the Lesson 24 state-label CSS overrides (white state labels, edge-label `foreignObject { overflow:visible }`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style. Question on the left (yellow label), value on the right. Fields matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `eval-event` blue). The value column shows name, rationale, and implementation. Click a row to expand.
5. **App UI** — the live sample.

## App UI tab

- **Submit form:** one multi-line field "Project brief" (product, audience, goal) + a Submit button. Submit POSTs `/api/briefs` and clears the field.
- **Live list:** campaigns streamed from `/api/campaigns/sse`, newest first. Each card shows the brief, a status pill (`RECEIVED … EVALUATED`, or `FAILED`), and timestamps.
- **Assembled campaign detail:** when status is `ASSEMBLED` or `EVALUATED`, the card expands to show the plan summary, the grounded claims with their sources, the strategy (positioning / channels / tactics), the content artifacts (taglines / posts), and — when `EVALUATED` — the self-eval score and notes.
- **No actions required from the user** beyond submitting; the pipeline runs autonomously.
