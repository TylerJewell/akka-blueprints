# UI Mockup — Lead Scoring Strategy

Single self-contained `src/main/resources/static-resources/index.html` (Lesson 17 — no `ui/`, no npm, no build step). Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Browser title: `<title>Akka Sample: Lead Scoring Strategy</title>`. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the governance.html anchor). Content fits the 1080px column with no horizontal scroll (Lesson 12).

## Five tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample type (not the flow), no subtitle. Four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — the four mermaid diagrams from `reference/architecture.md` (component graph, sequence, state machine, entity model) plus a component table.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`sanitizer` green, `eval-event` blue). The value column shows name, rationale, and implementation.
5. **App UI** — submit a lead; live lead list via SSE; each row expands to show score, eval result, and engagement strategy.

### App UI tab detail

- **Submit form** — fields: Company, Contact name, Form responses. Submit button POSTs `/api/leads`.
- **Lead list** — one row per lead from `/api/leads/sse`, showing company, status badge, and score (when present). Sorted by score descending, then by most recent.
- **Row expansion** — clicking a row reveals the intake profile, research brief, score rationale, the eval result (`evalScore` plus `evalFlags` — a non-`none` flag renders in a warning color), and the `engagementStrategy` once `status === "COMPLETE"`.

There are no Approve/Reject controls — the pipeline runs end to end without a human gate.

## Tab switching MUST be attribute-based (Lesson 26)

Switch by `data-tab` → `data-panel` attribute, never by NodeList index. Removed tabs must be deleted from the DOM, never hidden with `display:none` (hidden panels become zombies that take index positions and blank out the wrong tab).

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Each nav tab carries `data-tab="0..4"`; each panel carries the matching `data-panel="0..4"`. The two value sets line up exactly, with no extra panels in the DOM.

## Mermaid label overrides (Lesson 24)

The Architecture tab includes the CSS that forces `color`/`fill` white on every state-label DOM path and `overflow:visible` on edge-label `foreignObject` elements, plus `nodeTextColor` / `stateLabelColor` / `transitionLabelColor:#cccccc` in `mermaid.initialize`. Without these, state names render black-on-black and transition labels clip.
