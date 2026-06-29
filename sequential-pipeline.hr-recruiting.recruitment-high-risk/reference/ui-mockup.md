# UI mockup

Single self-contained `src/main/resources/static-resources/index.html`. Dark
theme, yellow accent, Instrument Sans, dot-grid background. Browser title:
`<title>Akka Sample: Recruitment Pipeline</title>`. Left nav with five tabs;
right content pane with one panel per tab. Content fits a 1080px column with no
horizontal scroll.

## Tabs

1. **Overview** — eyebrow "Overview" + headline (sample type, no subtitle), then
   four cards: Try it (`/akka:specify @SPEC.md`, then it auto-runs), How it works
   (source → sanitize → screen → match), Components (the inventory), API contract
   (the top-level surface).
2. **Architecture** — renders the four PLAN.md mermaid diagrams (component graph,
   sequence, state machine, ER) with the Lesson 24 CSS overrides so state labels
   are white and edge labels are not clipped.
3. **Risk Survey** — fetches `/api/metadata/risk-survey`, renders in the
   `matrix-card` / `matrix-row` style: question on the left (yellow label), answer
   on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render in muted
   italic as "To be completed by deployer."
4. **Eval Matrix** — fetches `/api/metadata/eval-matrix`, renders one
   `matrix-row` per control. The label column carries the control id and a colored
   mechanism pill (`guardrail` red, `sanitizer` green, `halt` red, `hotl` muted).
   The value column shows name, rationale, and implementation.
5. **App UI** — the live application (below).

## App UI tab

Form fields:
- `roleTitle` (text)
- `requirements` (textarea)
- `candidateCount` (number, 1–5)
- `forceBlockedDomain` (checkbox — routes sourcing at a non-allowlisted domain to
  exercise the guardrail)
- Submit button → `POST /api/requisitions`.

System control:
- A Halt / Resume toggle bound to `/api/system/halt` and `/api/system/resume`,
  showing current state from `/api/system/status`.

Candidate list (live via `/api/candidates/sse`, upsert by id):
- Each row shows role title, status badge, source handle, match score, and a
  short rationale once MATCHED.
- BLOCKED rows show the blocked domain; HALTED rows are visually muted; SANITIZED+
  rows show the redacted categories.

Monitoring panel:
- Reads `/api/monitoring`: evaluated, matched, rejected, blocked, average score.

## Tab-switching MUST be attribute-based (Lesson 26)

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

Removed tabs are deleted from the DOM, never hidden with `display:none` — a
hidden zombie panel still occupies a `querySelectorAll` index and blanks the
intended tab. There are exactly five `.nav-tab` and five `.tab-panel` elements,
with `data-tab`/`data-panel` values 0–4 lined up.
