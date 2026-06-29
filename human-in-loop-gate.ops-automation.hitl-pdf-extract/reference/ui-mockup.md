# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (anchor: `specs/vision/governance.html`). Browser title: `<title>Akka Sample: HITL PDF Data Extraction</title>`. Content wraps in a 1080px column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Includes the Lesson 24 mermaid theme variables and CSS overrides (state-label colour `#ffffff`, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` pairs (question label yellow on the left, value on the right). Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `sanitizer` blue, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — submit a document URL; live list of documents via SSE; per-document Approve/Reject buttons shown only when `status == PENDING_REVIEW` and `redactedFields` is present. Posted documents show the post target; rejected documents show the reason. Redacted field values are displayed to reviewers; raw field values are never shown.

## Tab switching MUST be attribute-based (Lesson 26)

Match nav tabs to panels by `data-tab` / `data-panel` attribute, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs are deleted from the DOM, not hidden with `display:none`. No zombie panels — a hidden panel still occupies a `querySelectorAll` index and breaks index-based switching.

## App UI fields and controls

- **Document URL input** — single text field + Submit button → `POST /api/extraction-request`.
- **Documents list** — one card per document: document URL, status badge, confidence bar, extracted field count, and (for `PENDING_REVIEW` documents) a reviewer-name field, a comment field, Approve and Reject buttons. Reject reveals a reason field. Fields shown in a key-value table using `redactedFields`.
- **Status badge** — `EXTRACTED` neutral, `PENDING_REVIEW` orange, `APPROVED` yellow, `POSTED` green, `REJECTED` muted red.
- **Confidence display** — numeric percentage plus a horizontal bar that fills yellow for high confidence, orange for low. Values below 85% render with a visual indicator that the document requires review.
