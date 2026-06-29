# UI mockup — meeting-to-tasks

A single self-contained `src/main/resources/static-resources/index.html` (Lesson 17 — no `ui/` folder, no npm build). Inline CSS + JS; runtime CDN imports for the markdown and YAML libs are fine. Browser title: `<title>Akka Sample: Meeting To Tasks</title>`. Visual style anchors to the dark / yellow-accent / Instrument Sans / dot-grid look. Content wraps in a 1080px column with no horizontal scroll (Lesson 12).

## Five tabs

1. **Overview** — eyebrow "Overview" + a headline naming the sample type (no subtitle), then four cards: Try it (just `/akka:specify @SPEC.md` then `/akka:build`), How it works (the sanitize → extract → approve → board/csv/notify pipeline), Components (the inventory), API contract (the top-level routes).
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Carries the Lesson 24 CSS overrides so state names and transition labels stay visible, and the `transitionLabelColor: #cccccc` theme variable.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style: question as a yellow label on the left, answer on the right. Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer").
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill (`guardrail` red, `sanitizer` green, `hitl` yellow). The value column shows name, rationale, and implementation.
5. **App UI** — the live surface.

## App UI tab

- **Submit form**: a `title` text input, a `rawNotes` textarea, and a Submit button. Submit POSTs `/api/meetings` and clears the form.
- **Live list**: meetings streamed over `/api/meetings/sse`, newest first. Each row shows the title, the status, the redaction count once sanitized, the extracted task list once extracted, and — once `COMPLETED` — the board URL, CSV path, and Slack reference.
- **Per-meeting controls**: when a meeting's status is `AWAITING_APPROVAL`, an Approve button and a Reject button (with a reason input) appear. Approve POSTs `/api/meetings/{id}/approve`; Reject POSTs `/api/meetings/{id}/reject`. The buttons disappear once the status leaves `AWAITING_APPROVAL`.

## Tab switching — MUST be attribute-based (Lesson 26)

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

Removed tabs are deleted from the DOM, never hidden with `display:none` — a hidden zombie panel takes an index position and silently breaks index-based switching. There must be exactly five `.tab-panel` sections, each with a `data-panel` value matching its nav tab's `data-tab`.
