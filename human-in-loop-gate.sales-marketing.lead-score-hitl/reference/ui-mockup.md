# UI Mockup — Lead Score HITL

Single self-contained `src/main/resources/static-resources/index.html` (Lesson 17 — no `ui/`, no npm, no build step). Inline CSS + JS; runtime CDN imports for markdown and YAML are acceptable. Browser title: `<title>Akka Sample: Lead Score HITL</title>`. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (the governance.html anchor). Content fits the 1080px column with no horizontal scroll (Lesson 12).

## Five tabs

1. **Overview** — eyebrow "Overview" + headline naming the sample type (not the flow), no subtitle. Four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — the four mermaid diagrams from `reference/architecture.md` (component graph, sequence, state machine, entity model) plus a component table.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style. Question on the left (yellow label), value on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted italic.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`hitl` yellow, `sanitizer` green, `guardrail` red). The value column shows name, rationale, and implementation.
5. **App UI** — submit a lead; live lead list via SSE; Approve/Reject buttons on `SHORTLISTED` leads.

### App UI tab detail

- **Submit form** — fields: Company, Contact name, Raw source. Submit button POSTs `/api/leads`.
- **Lead list** — one row per lead from `/api/leads/sse`, showing company, status badge, score (when present), and the score rationale. Sorted by score descending.
- **Review controls** — Approve and Reject buttons appear only when `status === "SHORTLISTED"`. Reject opens a one-line reason input. Approve POSTs `/api/leads/{id}/approve`; Reject POSTs `/api/leads/{id}/reject`.
- **Outreach preview** — when `status === "CONTACTED"`, the row expands to show `outreachSubject` and `outreachBody`.

## Tab switching MUST be attribute-based (Lesson 26)

Switch by `data-tab` → `data-panel` attribute, never by NodeList index. Removed tabs must be deleted from the DOM, never hidden with `display:none` (hidden panels become zombies that take index positions and blank out the wrong tab).

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

Each nav tab carries `data-tab="0..4"`; each panel carries the matching `data-panel="0..4"`. The two value sets line up exactly, with no extra panels in the DOM.

## Mermaid label overrides (Lesson 24)

The Architecture tab includes the CSS that forces `color`/`fill` white on every state-label DOM path and `overflow:visible` on edge-label `foreignObject` elements, plus `nodeTextColor` / `stateLabelColor` / `transitionLabelColor:#cccccc` in `mermaid.initialize`. Without these, state names render black-on-black and transition labels clip.
