# UI mockup — gemini-fullstack

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Gemini Fullstack Chat</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Gemini Fullstack <span class="accent">Chat</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **New conversation** to start a fresh thread, or pick a seeded example.
  3. Type a message and click **Send**.
  4. Watch the status chip transition from `AWAITING_REPLY` to `REPLY_RECORDED` as the agent reply arrives.
- Card **How it works**: one paragraph on message → workflow → agent → record → SSE push; one paragraph explaining that the conversation history is durable in an EventSourcedEntity and the browser receives updates over a single SSE connection per conversation.
- Card **Components**: rows per component (ConversationEntity, ConversationWorkflow, ChatAgent, ConversationContextBuilder, ConversationView, ChatEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, 1 Utility class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = assist-only` declaration and `oversight.human_in_loop = false` are the distinctive answers for a general chat assistant. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- The table is empty for this baseline — `controls: []`. Display a muted notice: "This baseline has no governance controls. A deployer targeting a regulated sector should add controls to `eval-matrix.yaml` before generating."

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Type a message. <span class="accent">Get a reply.</span>`. Subtitle: `One agent, durable history, live SSE updates.`
- Layout: two-column.
  - **Left column** — Conversation list panel.
    - A **New conversation** button at the top. A text input for the conversation title (defaults to "New conversation").
    - Below: one card per conversation, newest-first. Each card shows title, last-message preview (truncated to 60 chars), status chip, and age.
    - A "Load seeded example" link loads one of the three seeded conversations from `sample-events/seed-conversations.jsonl`.
  - **Right column** — Conversation thread panel (shown when a conversation is selected).
    - Thread header: conversation title + status chip.
    - Message list: a scrollable vertical list. User messages right-aligned with a blue background; agent replies left-aligned with a dark surface background. Each agent message shows a small token-count chip.
    - Status indicator: while the conversation is `AWAITING_REPLY`, a pulsing "Thinking..." row appears at the bottom of the thread.
    - Input area: a multi-line textarea and a yellow **Send** button. Send is disabled while `AWAITING_REPLY`. Pressing Enter without Shift submits.
- Status chip colours: CREATED=muted, ACTIVE=green, AWAITING_REPLY=yellow (pulsing), REPLY_RECORDED=green, FAILED=red.
- The SSE connection opens when the user selects a conversation and closes when they navigate away. At most one `EventSource` is open per conversation. The connection is not re-opened on every message — it is persistent for the lifetime of the right-pane view.
