# UI mockup — web-navigation-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Web Navigation Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Web Navigation <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Enter a task goal and starting URL, or click **Load seeded example** to fill both fields.
  3. Click **Start navigation**.
  4. Watch the session card progress through NAVIGATING → (AWAITING\_APPROVAL if a high-stakes action is encountered) → COMPLETED, with the action log and screenshots updating in real time.
- Card **How it works**: one paragraph on goal submission → screenshot capture → agent action loop → halt check; one paragraph on the three governance mechanisms (before-tool-call guardrail, operator halt switch, HITL approval gate).
- Card **Components**: rows per component (SessionEntity, HaltController, ScreenshotConsumer, NavigationWorkflow, WebNavigatorAgent, ActionGuardrail, HeadlessBrowserClient, SessionView, SessionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 HeadlessBrowserClient as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `human_on_loop: true` and `operator_halt_available: true` declarations are prominent. `decisions.authority_level = fully-autonomous` and `high_stakes_actions_require_hitl_approval: true` are the distinctive answers. `TO_BE_COMPLETED_BY_DEPLOYER` fields render as muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, HITL1). ID badges coloured: G1 red (guardrail), H1 orange (halt), HITL1 yellow (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a task. <span class="accent">Watch the agent navigate.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Task submission panel + live session list.
    - Submission panel: `Task goal` textarea, `Starting URL` text input (pre-filled with `https://doc.akka.io`), `Max steps` number input (default 20), `Submitted by` text input, a `Load seeded example` link that fills goal + URL, and a yellow `Start navigation` button.
    - Global halt row: a red `Halt all sessions` button (calls `POST /api/halt`) and a muted `Clear halt` link (calls `DELETE /api/halt`). The halt state is shown as a persistent banner when active.
    - Live session list below: one card per session, newest-first. Each card shows status pill, age, and the first line of the task goal.
  - **Right column** — Selected-session detail.
    - Header: status pill + `sessionId` + goal text.
    - Current page section: URL label + screenshot thumbnail (most recent `screenshotPath`).
    - HITL approval panel (visible only when `status = AWAITING_APPROVAL`): proposed action type, target selector/URL, agent rationale, and two buttons — green `Approve` and red `Reject`. The panel is bordered orange to draw attention.
    - Action log table: columns step #, action type (coloured chip), target, executed (yes/blocked), rationale, timestamp.
    - Outcome section (visible when `status = COMPLETED`): success badge, `taskResult` string, steps used.
- Status pill colours: STARTING=muted, NAVIGATING=yellow, AWAITING\_APPROVAL=orange, COMPLETED=green, HALTED=red, FAILED=red, REJECTED=muted.
- Action type chip colours: CLICK=blue, TYPE=purple, SCROLL=muted, NAVIGATE=blue, COMPLETE=green, REJECT\_TASK=red.
- Blocked action rows are rendered with a strikethrough and a muted red `blocked` badge showing the block reason.
