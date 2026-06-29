# UI mockup — with Signals & Queries

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: with Signals &amp; Queries</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Signal-Driven <span class="accent">Chat</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Type a session name and an initial prompt, then click **Start session**.
  3. When the first reply appears, type a follow-up in the **Add turn** field and click **Send**.
  4. Click **Pause**, type an operator note, then click **Resume** to see the agent incorporate your note.
- Card **How it works**: one paragraph on start → signal → agent → reply loop; one paragraph on the two governance mechanisms (inbound-signal guardrail, operator HITL pause/resume).
- Card **Components**: rows per component (`ChatSessionEntity`, `ChatSessionWorkflow`, `ChatAgent`, `SignalValidator`, `SessionView`, `ChatEndpoint`, `AppEndpoint`) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `human_in_loop: true` and `operator_can_pause_and_resume: true` declarations are filled and prominent. Most other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, G1). ID badges coloured: H1 purple (hitl), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Start a session. <span class="accent">Signal it mid-flight.</span>`. Subtitle: `One agent, signals and queries, operator HITL.`
- Layout: two-column.
  - **Left column** — Session list + new-session form.
    - New-session form at the top: `Session name` text input, `Initial prompt` textarea, `Submitted by` text input, and a yellow `Start session` button.
    - Live list below: one card per session, newest-first. Each card shows status pill, turn count badge, session name, age. Clicking a card selects it in the right pane.
  - **Right column** — Selected-session detail.
    - Header: status pill + session name + turn count + `Query State` button (fires `GET /api/sessions/{id}/query` and refreshes the pane without adding a turn).
    - Conversation thread: a scrollable list of turns in chronological order. Each turn shows the source badge (`USER` / `OPERATOR` / `SYSTEM`), the prompt text, the reply text, and the elapsed time between `sentAt` and `repliedAt`.
    - Operator panel (visible in all non-CLOSED, non-FAILED states): a horizontal row of signal buttons — **Pause** (disabled in PAUSED), **Resume** (disabled unless PAUSED), **Close**. When PAUSED, a `Corrective note` textarea and a `Send Resume` button appear.
    - Add-turn panel (visible in ACTIVE state only): a single-line `Your prompt` input and a `Send` button that fires `PUT /api/sessions/{id}/signal/turn`. If the signal is rejected (400), the error detail (`rejectedCheck`) appears inline beneath the input in red.
- Status pill colours: STARTING=muted, ACTIVE=green, PAUSED=yellow, CLOSING=blue, CLOSED=muted, FAILED=red.
- Turn source badge colours: USER=blue, OPERATOR=purple, SYSTEM=muted.
- Guardrail-rejected signal errors are shown inline in the add-turn panel — never as a browser `alert()`.
