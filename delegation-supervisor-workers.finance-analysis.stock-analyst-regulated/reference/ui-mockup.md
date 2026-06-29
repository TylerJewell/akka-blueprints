# UI mockup — Stock Analysis Team

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`), YAML (`js-yaml`), and mermaid are acceptable. Browser title: `<title>Akka Sample: Stock Analysis Team</title>`. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the governance.html anchor). Content wraps in `.wrap{ max-width:1080px }` with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample (delegation supervisor/workers, finance analysis), no subtitle. Four cards: Try it (`/akka:specify @SPEC.md` then `/akka:build`), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams (component graph, sequence, state machine, ER). Mermaid initialized with the Akka theme variables AND the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style: question (yellow label) left, answer right. Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same style. Each label carries a colored mechanism pill: `guardrail` red, `hotl` muted, `sanitizer` green, `eval-periodic` blue. The value column shows name, rationale, and implementation.
5. **App UI** — the live surface (below).

## App UI tab

- **Submit form** — a text input for a ticker and a Submit button. POSTs `/api/analyses`.
- **Analyses list** — live via `GET /api/analyses/sse`. Each row shows ticker, status badge, recommendation call (when present), and a drift flag when set.
- **Row detail** — expands to show summary, rationale, disclaimer, and sanitized notes.
- **Review actions** — when status is `ISSUED_PENDING_REVIEW`, the row shows a Clear button and a Retract button (each posts the reviewer + note). The buttons disappear once the analysis is `CLEARED` or `RETRACTED`.

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

Removed tabs are deleted from the DOM, never hidden with `display:none` — a hidden panel still occupies a `querySelectorAll` position and produces a blank App UI tab.
