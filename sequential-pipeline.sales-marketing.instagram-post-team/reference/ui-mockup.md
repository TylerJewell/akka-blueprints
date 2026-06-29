# UI mockup

A single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the governance.html anchor). Content wraps in a 1080px column with no horizontal scroll. Browser title: `<title>Akka Sample: Instagram Post Team</title>`.

## Tab switching MUST be attribute-based (Lesson 26)

The left-nav has five tabs. Switching matches by `data-tab` / `data-panel` attribute, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs are deleted from the DOM, never hidden with `display:none`. No zombie panels.

## Tabs

### 1. Overview (`data-panel="overview"`)
Eyebrow "Overview" + headline naming the sample type (sequential pipeline Instagram post team), no subtitle. Four cards: Try it (`/akka:specify @SPEC.md`, then it auto-builds — no env-var export block), How it works (caption → image prompt → brand/safety check → ready), Components (the inventory), API contract (the top-level surface).

### 2. Architecture (`data-panel="architecture"`)
Renders the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, entity model). Apply the Lesson 24 theme variables and CSS overrides so state-box labels are white and edge labels are not clipped (`overflow:visible` on edge-label `foreignObject`, `transitionLabelColor: #cccccc`). A click-to-expand component table follows.

### 3. Risk Survey (`data-panel="risk-survey"`)
Renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Each answer is a row: question label (yellow) on the left, value on the right. Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer").

### 4. Eval Matrix (`data-panel="eval-matrix"`)
Renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `ci-gate` pale yellow). The value column shows name, rationale, and implementation. Click a row to expand.

### 5. App UI (`data-panel="app-ui"`)
The live sample.

- **Submit form:** a single text input ("Creative brief") and a Submit button → `POST /api/briefs`.
- **Post list:** subscribes to `GET /api/posts/sse`; renders one card per post showing brief, status pill, caption, hashtags, and image prompt once present.
- **Blocked posts:** when status is `BLOCKED`, the card shows the `blockReason` in place of the ready content.
- **Status visualisation:** color-coded status pill per post (composing/prompting neutral, ready green, blocked/failed red).
