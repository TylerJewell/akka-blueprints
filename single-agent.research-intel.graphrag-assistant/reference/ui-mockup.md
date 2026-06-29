# UI mockup — graphrag-assistant

One self-contained `src/main/resources/static-resources/index.html`. No `ui/`
folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown, YAML,
and mermaid are acceptable. Browser title:
`<title>Akka Sample: GraphRAG Assistant</title>`. Visual style: dark background,
yellow accent, Instrument Sans, dot-grid — matching `governance.html`. Content
column max-width 1080px, no horizontal scroll (Lesson 12).

## Tabs

1. **Overview** — eyebrow ("Overview") + headline naming the sample type (no
   subtitle), then four cards: Try it (the single `/akka:build` step, no env-var
   export block), How it works (receive → retrieve → ground-check → answer),
   Components (the seven from SPEC.md Section 4), API contract (the top-level
   surface).
2. **Architecture** — the four mermaid diagrams from `PLAN.md`, rendered with the
   Akka theme variables and the Lesson 24 CSS overrides (state-label color, edge-
   label `foreignObject { overflow:visible }`, `transitionLabelColor:#cccccc`).
3. **Risk Survey** — `/api/metadata/risk-survey` rendered in `matrix-card` /
   `matrix-row` style: yellow question label left, value right. Values matching
   `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted-italic as "To be completed by
   deployer".
4. **Eval Matrix** — `/api/metadata/eval-matrix` in the same `matrix-card` /
   `matrix-row` style. Each control's label column carries a colored mechanism
   pill: `guardrail` red, `sanitizer` green. The value column shows name,
   rationale, and implementation.
5. **App UI** — an ask box (textarea + "Ask" button → `POST /api/ask`); a live
   list of queries via `GET /api/queries/sse`. Each query row shows the question,
   the status badge (`RECEIVED` / `ANSWERED` / `BLOCKED`), the chosen `scope`
   (local/global) pill, the answer text, a grounded indicator, and the citation
   doc-ids. `BLOCKED` rows show the `blockedReason`.

## Tab switching MUST be attribute-based (Lesson 26)

Switch tabs by matching `data-tab` (nav) to `data-panel` (section), never by
NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs must be deleted from the DOM, not hidden with `display:none` — a
hidden zombie panel takes an index slot and blanks the wrong tab. There are
exactly five `.tab-panel` sections, one per tab above.
