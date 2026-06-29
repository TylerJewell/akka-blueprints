# UI mockup

A single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the governance.html anchor). Content wraps in a 1080px column with no horizontal scroll. Browser title: `<title>Akka Sample: Social Post Team</title>`.

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
Eyebrow "Overview" + headline naming the sample type (sequential pipeline social post team), no subtitle. Four cards: Try it (`/akka:specify @SPEC.md`, then it auto-builds — no env-var export block), How it works (research → compose → brand check → approve → publish), Components (the inventory), API contract (the top-level surface).

### 2. Architecture (`data-panel="architecture"`)
Renders the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, entity model). Apply the Lesson 24 theme variables and CSS overrides so state-box labels are white and edge labels are not clipped (`overflow:visible` on edge-label `foreignObject`, `transitionLabelColor: #cccccc`). A click-to-expand component table follows.

### 3. Risk Survey (`data-panel="risk-survey"`)
Renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Each answer is a row: question label (yellow) on the left, value on the right. Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer").

### 4. Eval Matrix (`data-panel="eval-matrix"`)
Renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `hitl` yellow, `eval-periodic` blue, `ci-gate` pale yellow). The value column shows name, rationale, and implementation. Click a row to expand.

### 5. App UI (`data-panel="app-ui"`)
The live sample.

- **Submit form:** a single text input ("Creative concept") and a Submit button → `POST /api/concepts`.
- **Post list:** subscribes to `GET /api/posts/sse`; renders one card per post showing concept, status pill, caption, hashtags, and visual brief once present.
- **Per-post actions:** when status is `AWAITING_APPROVAL`, show Approve and Reject buttons. Approve posts `{ approvedBy, comment }`; Reject opens a reason field and posts `{ rejectedBy, reason }`. Buttons disappear once the post leaves `AWAITING_APPROVAL`.
- **Status visualisation:** color-coded status pill per post (researching/composing neutral, awaiting-approval yellow, published green, rejected/escalated red).
