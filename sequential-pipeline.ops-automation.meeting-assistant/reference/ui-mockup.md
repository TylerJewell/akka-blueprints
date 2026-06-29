# UI mockup

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown, YAML, and mermaid are acceptable. Browser title: `<title>Akka Sample: Meeting Assistant</title>`. Visual style: dark background, yellow accent, Instrument Sans, dot-grid. Content wraps in `.wrap { max-width:1080px }` with no horizontal scroll.

## Tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample type (sequential pipeline), no subtitle. Four cards: **Try it** (one line: run `/akka:build`, open the App UI tab, paste notes), **How it works** (redact → extract → dispatch), **Components** (the component inventory), **API contract** (the top endpoints).
2. **Architecture** — the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, entity model), each in a `.diagram-card`. Includes the Lesson 24 CSS overrides and theme variables so state-diagram labels render white and edge labels are not clipped.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style. Each answer is a row: question on the left (yellow label), value on the right. Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer").
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`sanitizer` green, `guardrail` red, `eval-periodic` blue, `ci-gate` pale yellow, `halt` red). The value column shows name, rationale, and implementation.
5. **App UI** — the live sample.

## App UI tab

- A textarea labelled "Meeting notes" and a smaller "Title" input.
- A **Submit** button. POSTs to `/api/meetings`; clears the textarea on success.
- A live list of meetings via `EventSource` on `/api/meetings/sse`. Each card shows: title, a status chip (`RECEIVED`/`SANITIZED`/`EXTRACTED`/`DISPATCHED`/`FAILED`), the redaction count once sanitized, the summary and action-item list once extracted (each item showing title, assignee hint, due hint, and Trello card link once dispatched), the Slack message timestamp once dispatched, and the failure reason if `FAILED`.

## Tab-switching MUST be attribute-based (Lesson 26)

Switch tabs by matching `data-tab` on the nav button to `data-panel` on the section — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs must be **deleted** from the DOM, not hidden with `display:none`. A hidden panel still appears in `querySelectorAll('.tab-panel')` and an index-based switcher would activate the wrong panel, leaving the App UI blank.
