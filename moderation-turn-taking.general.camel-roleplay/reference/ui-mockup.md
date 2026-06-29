# UI mockup

The generated UI is a single self-contained `src/main/resources/static-resources/index.html` (Lesson 17): inline CSS and JS, runtime CDN imports for markdown and YAML rendering only, no `ui/` folder, no build step. Visual style follows `specs/vision/governance.html` — dark background, yellow accent, Instrument Sans, dot-grid. Browser title:

```html
<title>Akka Sample: CAMEL Role-Play Collaboration</title>
```

A left-nav with five tabs; the content column is capped at 1080px with no horizontal scroll (Lesson 12).

## Tab 1 — Overview

Eyebrow `Overview`, headline naming the sample type (a moderated role-play collaboration), **no subtitle**, then four cards:

- **Try it** — the one command: `/akka:build`. No env-var export block.
- **How it works** — three sentences: a task request starts a workflow; the Coordinator alternates a User agent and an Assistant agent for up to twelve rounds; the Coordinator declares the task solved with a final answer, or an impasse when convergence is not reached.
- **Components** — the count: three agents, one workflow, two event-sourced entities, one key-value entity, one view, two consumers, two timed actions, two endpoints.
- **API contract** — the top-level paths from [`api-contract.md`](api-contract.md).

Rendered from `/api/metadata/readme` as markdown.

## Tab 2 — Architecture

The four mermaid diagrams from [`../PLAN.md`](../PLAN.md): component graph, interaction sequence, state machine, entity model. Each in a `.diagram-card`. Below them, a click-to-expand component table listing every component, its Akka primitive, and its role.

The mermaid block must carry the Lesson 24 fixes so the state-machine diagram is legible: explicit `color`/`fill` on the state-label DOM paths, `overflow:visible` on every edge-label `foreignObject`, and `nodeTextColor` / `stateLabelColor` / `transitionLabelColor: #cccccc` in `mermaid.initialize`.

## Tab 3 — Risk Survey

Renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style: the question on the left as a yellow label, the answer on the right. Values matching `/TO_BE_COMPLETED_BY_DEPLOYER/i` render muted and italic as "To be completed by deployer", so the deployer-specific surface is visually distinct from the pre-filled sample answers.

## Tab 4 — Eval Matrix

Renders `/api/metadata/eval-matrix` in the same `matrix-card` / `matrix-row` style. The label column carries the control id plus a colored mechanism pill — `guardrail` red, `eval-event` blue, `ci-gate` pale yellow, `halt` red. The value column shows the control name, rationale, and implementation note. Four controls: G1, E1, A1, HT1. Clicking a row expands its rationale.

## Tab 5 — App UI

The live interaction.

**Start form** — three fields and a button:

| Field | Type | Note |
|---|---|---|
| Task description | text | what the collaboration must solve |
| Assistant role | text | role label for the AI Assistant (for example "Python developer") |
| User role | text | role label for the AI User (for example "software engineer giving requirements") |
| **Start** | button | POSTs `/api/collaborations`, clears the form |

**Operator control** — a Halt / Resume toggle bound to `/api/system/halt`, `/api/system/resume`, and `/api/system/status`. When halted, a banner reads "New collaborations paused" and the Start button is disabled; in-flight collaborations keep running.

**Collaborations list** — fed by `GET /api/collaborations/sse`, keyed by `id`, newest first. Each row shows:

- the task description (truncated if long), the assistant and user role labels, and a status chip (`COLLABORATING` yellow, `CONCLUDED` green for `SOLVED` / muted for `IMPASSE`, `ESCALATED` red);
- the current round out of twelve;
- an inline-expandable turn history — one line per `TurnLine`: round, agent (Assistant or User), intent, and a preview of the message, so the problem-solving process is traceable;
- on conclusion, the outcome, the solution summary and final answer (when `SOLVED`), and the evaluator score with its notes.

No approve/reject buttons — this pattern has no human gate; the only operator action is halt/resume.

## Tab-switching MUST be attribute-based (Lesson 26)

Index-based tab switching breaks when a removed tab leaves a hidden zombie panel in the DOM: the nav tab's index no longer lines up with the panel's index, and the user clicks a tab and sees a blank panel. Both fixes are mandatory in the generated `index.html`:

1. **Delete removed tabs from the DOM.** Never hide a retired panel with `display:none` and leave the `<section>` in place. If a tab no longer exists, its `<section class="tab-panel">` is removed entirely.

2. **Switch by `data-tab` → `data-panel` attribute, never by NodeList index.** Each nav tab carries `data-tab="0..4"` and each panel carries the matching `data-panel="0..4"`:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
document.querySelectorAll('.nav-tab').forEach(t =>
  t.addEventListener('click', () => switchTab(t.dataset.tab)));
```

The five panels are `data-panel="0"` Overview, `"1"` Architecture, `"2"` Risk Survey, `"3"` Eval Matrix, `"4"` App UI — matching the five `data-tab` values in left-nav order.
