# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the `governance.html` look). Content wraps in a 1080px column with no horizontal scroll. Browser title: `<title>Akka Sample: Industry Research Team</title>`.

## Tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample type (not the flow), no subtitle. Four cards: Try it (`/akka:build`, then open the listening URL), How it works (plan → analyze → synthesize → sanitize in one line), Components (the inventory from SPEC.md Section 4), API contract (the top-level surface).
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams (component graph, sequence, state machine, ER) with the Lesson 24 state-label CSS overrides and theme variables.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Each answer is a row: question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic ("To be completed by deployer").
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `sanitizer` green). The value column shows name, rationale, and implementation notes.
5. **App UI** — a request input box + Submit button (`POST /api/research-request`); a live list of briefs via `GET /api/briefs/sse`. Each brief card shows the title, status, sector chips, the grounding score, and — when `COMPLETED` — the sanitized body. A `BLOCKED` card shows the block reason in place of a body.

## Tab switching MUST be attribute-based (Lesson 26)

Switch by `data-tab` → `data-panel` attribute, never by NodeList index. Nav tabs carry `data-tab="0".."4"`; panels carry the matching `data-panel`. Removed tabs are deleted from the DOM, never hidden with `display:none` — a hidden zombie panel shifts NodeList indices and blanks the App UI tab.

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```
