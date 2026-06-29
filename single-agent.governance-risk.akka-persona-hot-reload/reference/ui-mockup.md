# UI mockup — persona-hot-reload

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: PersonaHotReload</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. Canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Persona<span class="accent">HotReload</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded persona fixture (Assistant / Escalation / Restricted) from the dropdown.
  3. Click **Push persona**.
  4. Watch the card transition through VALIDATING → ACTIVATING → ACTIVE → REVALIDATING → MONITORED. Send a chat message to confirm the new persona is live.
- Card **How it works**: one paragraph on push → gate → activate → revalidate → monitor; one paragraph on the three governance mechanisms (configuration gate, behavioral revalidation, monitoring window).
- Card **Components**: rows per component (PersonaEntity, PersonaChangeConsumer, PersonaWatchWorkflow, PersonaAgent, PersonaValidator, BehavioralRevalidator, PersonaView, PersonaEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Validator + 1 Revalidator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = direct-action` declaration is prominent (this is unusual — direct-action means a persona push immediately affects all conversations). `oversight.human_on_loop = true` and `oversight.human_in_loop = false` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (C1, E1, H1). ID badges coloured: C1 red (ci-gate), E1 blue (eval-event), H1 orange (hotl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Push a persona. <span class="accent">See it live.</span>`. Subtitle: `One agent, governed across every persona swap.`
- Layout: two-column.
  - **Left column** — Persona push panel + change log.
    - Push panel: dropdown `Persona fixture` (Assistant / Escalation / Restricted / custom), `Agent role` text input, `Agent goal` text input, `Agent instructions` textarea, `Agent model` text input (optional), `Pushed by` text input, and a yellow `Push persona` button. Selecting a fixture from the dropdown pre-fills the four persona fields. Selecting "custom" clears them.
    - Change log below: one card per persona change, newest-first. Each card shows status pill, revalidation badge (when available), fixture label or "custom", pushedBy, age.
  - **Right column** — Two panels stacked vertically.
    - **Selected-change detail panel** (top):
      - Header: status pill + revalidation badge + changeId (truncated).
      - Persona fields: Role, Goal, Instructions (truncated at 200 chars with expand), Model.
      - Revalidation section: overall status badge (PASS/PARTIAL/FAIL), then a probe table — columns: probeId, question (truncated), outcome chip, rationale.
      - Monitoring window section: open/closed indicator, forwardedCount, elapsed or closed-at timestamp. Orange banner if window is currently open.
      - Rejection reason (if `REJECTED`): shown in a red box.
    - **Chat panel** (below): a text input + `Send` button. On submit, POSTs to `/api/personas/chat`. Response appears below in a message bubble. The current changeId is shown above the input: "Active persona: `{changeId truncated}`". Chat history is local (no persistence).
- Monitoring banner: when the most-recent change has `monitoring.open == true`, an orange banner spans the top of the right column: "Monitoring window open — {elapsed}s elapsed — {forwardedCount} responses forwarded to operator channel."
- Status pill colours: VALIDATING=muted, REJECTED=red, ACTIVATING=yellow, ACTIVE=blue, REVALIDATING=yellow, MONITORED=green, FAILED=red.
- Revalidation badge colours: PASS=green, PARTIAL=yellow, FAIL=red.
- Gate-result indicator on REJECTED cards: red inline label with the rejection reason.
