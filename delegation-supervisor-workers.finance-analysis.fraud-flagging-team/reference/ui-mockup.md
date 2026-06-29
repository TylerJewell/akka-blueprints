# UI mockup — fraud-flagging-team

A single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build (Lesson 17). Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Browser title: `<title>Akka Sample: Fraud Flagging Team</title>`. Visual style anchor: dark background, yellow accent, Instrument Sans, dot-grid (the `governance.html` style). Content wraps in `.wrap{ max-width:1080px }` with no horizontal scroll (Lesson 12).

## Tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample type (not the flow), no subtitle. Four cards: Try it (`/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — the four mermaid diagrams from `PLAN.md` (component graph, sequence, state machine, ER). Render with the Akka theme variables and the Lesson 24 CSS overrides so state labels are white-on-dark and edge labels are not clipped.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Question on the left (yellow label), answer on the right. Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer").
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `sanitizer` green, `eval-event` blue, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — the live interaction.

## App UI tab

- **Submit form:** fields `customerId` (text), `amount` (number), `transactionRef` (text), `memo` (text). Submit button POSTs `TransactionInput` to `/api/cases`.
- **Live case list:** streamed from `/api/cases/sse`. Each card shows customer id, amount, status badge, fraud score, risk tier, compliance result, and the supervisor rationale once present.
- **Per-case actions:** when status is `FLAGGED`, show Confirm and Dismiss buttons. Confirm POSTs `/api/cases/{id}/confirm`; Dismiss opens a small note field then POSTs `/api/cases/{id}/dismiss`. Buttons disappear once the case leaves `FLAGGED`.
- **Status badges:** ANALYZING (muted), FLAGGED (yellow), CLEARED (green), CONFIRMED (blue), ACTIONED (green), DISMISSED (muted), ESCALATED (red).

## Tab-switching MUST be attribute-based (Lesson 26)

Switch tabs by matching `data-tab` on the nav element to `data-panel` on the section, never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Removed tabs must be deleted from the DOM, not hidden with `display:none` — a hidden zombie panel takes an index slot and breaks index-based switching. There are exactly five `.tab-panel` sections, one per tab above.
