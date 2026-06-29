# UI mockup — Fair CV Matcher

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Browser title: `<title>Akka Sample: Fair CV Matcher</title>`. Visual style: dark, yellow accent, Instrument Sans, dot-grid background. Content wraps in a 1080px column with no horizontal scroll (Lesson 12).

Five left-nav tabs.

## 1. Overview

Eyebrow "Overview" + headline naming the sample type (sequential pipeline for fair CV matching), no subtitle. Then four cards: Try it (one line: run `/akka:build`), How it works (extract → sanitize → match → review), Components (the inventory), API contract (top-level paths).

## 2. Architecture

The four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, entity model), each in a `.diagram-card`, plus a click-to-expand component table. Mermaid is initialized with the Akka theme variables and the Lesson 24 CSS overrides: white state-label colour on every label DOM path, and `overflow:visible` on every edge-label `foreignObject` so transition labels are not clipped.

## 3. Risk Survey

Renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style. Question on the left as a yellow label, answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic so deployer-specific fields are visually distinct.

## 4. Eval Matrix

Renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style, one row per control. The label column carries the control id plus a coloured mechanism pill: `sanitizer` green, `eval-periodic` blue, `hotl` muted, `ci-gate` pale yellow. The value column shows name, rationale, and implementation notes.

## 5. App UI

The live sample.

- **Submit form:** a textarea for `cvText`, a small editable list of postings (jobId, jobTitle, requirements), a `slice` text field, and a Submit button posting to `/api/candidates`.
- **Candidate list:** streamed over `/api/candidates/sse`, upserted by `id`, newest first. Each row shows id, status badge, and slice.
- **Candidate detail:** expanding a row shows the redaction summary (each `RedactedSignal` as field → category), the ranked `matches` (jobTitle, score bar, rationale), and the profile.
- **Review action:** on `MATCHED` candidates, a Review control posts `{ reviewedBy, note }` to `/api/candidates/{id}/review`; the row moves to `REVIEWED`.
- **Fairness strip:** a small panel fed by `/api/fairness` listing each slice's selection rate and parity ratio, flagged slices highlighted.

## Tab-switching MUST be attribute-based (Lesson 26)

Tabs carry `data-tab="0..4"`; panels carry `data-panel="0..4"`. Switching matches `data-tab` to `data-panel` by attribute value, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs are deleted from the DOM, never hidden with `display:none`. No zombie panels may remain, or the index mapping breaks and a tab renders blank.
