# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build. Inline CSS + JS; runtime CDN imports for markdown (`marked`) and YAML (`js-yaml`) are acceptable. Visual style: dark background, yellow accent, Instrument Sans, dot-grid (anchor: `specs/vision/governance.html`). Browser title: `<title>Akka Sample: Ambient Email Assistant</title>`. Content wraps in a 1080px column with no horizontal scroll.

## Five tabs

1. **Overview** — eyebrow "Overview" + headline (the sample type, not the flow), no subtitle, then four cards: Try it (just `/akka:build`, no env-var export block), How it works, Components, API contract.
2. **Architecture** — renders the four `PLAN.md` mermaid diagrams. Includes the Lesson 24 mermaid theme variables and CSS overrides (state-label colour `#ffffff`, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in `matrix-card` / `matrix-row` pairs (question label yellow on the left, value on the right). Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted italic.
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one `matrix-row` per control. The label column carries the control id plus a coloured mechanism pill (`hitl` yellow, `guardrail` red, `sanitizer` blue, `eval-event` purple). The value column shows name, rationale, and implementation.
5. **App UI** — submit an incoming email (sender, subject, body fields); live list of threads via SSE; per-thread Approve/Dismiss buttons shown only when `status == ACTION_DRAFTED`. Approved threads show the action reference; dismissed threads show the dismiss note; meeting threads show the proposed time and attendees.

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

- **Incoming email form** — sender text field, subject text field, body textarea, Submit button → `POST /api/email-threads`.
- **Threads list** — one card per thread: subject, status badge, category + urgency pills, and (for `ACTION_DRAFTED` threads) the draft preview, an approver-name field, a note field, Approve and Dismiss buttons. Dismiss reveals a reason field.
- **Status badge** — `RECEIVED` neutral, `TRIAGED` blue, `ACTION_DRAFTED` yellow, `APPROVED` light green, `REPLY_SENT` green, `MEETING_SCHEDULED` teal, `DISMISSED` muted red.
- **Category pill** — `inquiry` grey, `action-request` orange, `meeting-request` teal, `notification` muted, `spam` red.
- **Draft preview** — for reply drafts, shows `draftSubject` + `draftBody`; for meeting proposals, shows `proposedMeetingTitle`, `proposedMeetingTime`, and `proposedAttendees`.
