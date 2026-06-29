# UI mockup — bidi-demo

Five-tab structure. Browser title: `<title>Akka Sample: BidiDemo</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Bidi<span class="accent">Demo</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **Open channel** to create a new bidirectional channel.
  3. Type a message and click **Send**.
  4. Watch response frames stream in one sentence at a time until `done == true`.
- Card **How it works**: one paragraph on open-channel → send-message → frame-streaming → turn-complete; one paragraph on the guardrail mechanism (FrameGuardrail validates every frame list before any frame reaches the SSE wire).
- Card **Components**: rows per component (ChannelEntity, MessageForwarder, ChannelWorkflow, ChannelAgent, FrameGuardrail, ChannelView, ChannelEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `decisions.authority_level = interactive` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Open a channel. <span class="accent">Stream a reply.</span>`. Subtitle: `One agent, frame by frame.`
- Layout: two-column.
  - **Left column** — Channel list + open-channel button.
    - **Open channel** button at the top. Clicking it shows a small inline form: channel name text input (auto-generated placeholder like `channel-${uuid.slice(0,6)}`), turn budget number input (default 10), opened-by text input, and a **Create** button. Submitting POSTs to `/api/channels`.
    - Channel list below: one card per channel, newest-first. Each card shows: status pill, channel name, turn count / budget (e.g. `3 / 10`), age. Clicking a card selects it in the right pane.
  - **Right column** — Selected channel detail.
    - Channel header: status pill + channel name + turn budget indicator.
    - Turn history: a scrollable list of turn cards (newest at the bottom). Each turn card shows:
      - Inbound message section: the user message text and sender.
      - Response frames section: frames appear one by one as SSE events arrive. Each frame shows its `frameIndex`, the `content` text, and a pulsing dot while the turn is still `PROCESSING`. When `done == true`, the pulsing dot stops.
      - Turn-status badge: `PROCESSING` (yellow), `COMPLETE` (green), `FAILED` (red).
    - Message input at the bottom: a textarea and a **Send** button. The button is disabled when the channel status is `CLOSED` or `FAILED`, or while the current turn is `PROCESSING`. Submitting POSTs to `/api/channels/{id}/messages`.
- Channel status pill colours: OPEN=blue, ACTIVE=green, CLOSED=muted, FAILED=red.
- Turn status badge colours: PROCESSING=yellow, COMPLETE=green, FAILED=red.
- When the channel reaches `CLOSED`, a banner replaces the input area: "Turn budget exhausted. Open a new channel to continue."
- The SSE connection to `/api/channels/{id}/sse` is opened when the user selects a channel and stays open until the channel status reaches a terminal state (`CLOSED` or `FAILED`). Frame events update the right pane in real time without any polling.
