# UI mockup — `recruitment-team`

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm (Lesson 17). Inline CSS + JS; runtime CDN imports for markdown and YAML are fine. Browser title `<title>Akka Sample: Recruitment Team</title>`. Visual style: dark, yellow accent, Instrument Sans, dot-grid background (governance.html anchor). Content wraps in `.wrap{ max-width:1080px }` with no horizontal scroll (Lesson 12).

## Five tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample type (no subtitle). Four cards: Try it (`/akka:build`), How it works, Components, API contract.
2. **Architecture** — the four mermaid diagrams from `PLAN.md` with the Lesson 24 CSS overrides applied (state-label colour forced white, edge-label `foreignObject` `overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic ("To be completed by deployer").
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same style. The label column carries the control id plus a colored mechanism pill: `sanitizer` green, `eval-event`/`eval-periodic` blue, `hitl` yellow. The value column shows name, rationale, and implementation.
5. **App UI** — the live surface. A form with a role-id field and a resume textarea plus a Submit button (POST `/api/applications`). A candidate list streamed via SSE; each row shows status, match score, eval score, the supervisor recommendation, and a drift badge when `driftFlagged` is true. Candidates in `AWAITING_DECISION` show Approve and Reject buttons (Reject opens a reason field).

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

Removed tabs are deleted from the DOM, never hidden with `display:none` — a hidden panel still occupies a `querySelectorAll` index and would strand the App UI panel blank. Each `.nav-tab` `data-tab` value matches exactly one `.tab-panel` `data-panel` value.
