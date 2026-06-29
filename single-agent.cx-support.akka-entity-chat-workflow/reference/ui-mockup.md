# UI mockup — entity-workflow-chat

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Entity Workflow Chat</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Entity Workflow <span class="accent">Chat</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **New conversation** and enter a topic (or load a seeded scenario).
  3. Type a message and click **Send**.
  4. Watch the turn cycle through `AGENT_THINKING` → `OPEN`; continue across multiple turns.
- Card **How it works**: one paragraph on start → message received → sanitize → agent reply → record; one paragraph on the compaction mechanism (token budget exceeded → ContextCompactor → summary → context window bounded).
- Card **Components**: rows per component (ConversationEntity, MessageSanitizer, ChatWorkflow, ConversationAgent, ContextCompactor, ConversationView, ChatEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 ContextCompactor as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = advisory-response` and `oversight.human_in_loop = true` are the distinctive answers. `oversight.reviewer_drives_every_turn = true` highlights the structural HITL guarantee. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, H1). ID badges coloured: S1 green (sanitizer), H1 purple (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Start a conversation. <span class="accent">Send a message.</span>`. Subtitle: `One agent, two governance mechanisms — every turn driven by the customer.`
- Layout: two-column.
  - **Left column** — New conversation panel + live conversation list.
    - New conversation panel: `Topic` text input, `User ID` text input (pre-filled with `demo-user`), a dropdown `Seeded scenario` (Billing dispute / Password reset / Order status / custom), and a yellow `New conversation` button.
    - Live list below: one card per conversation, newest-first. Each card shows status pill, topic, age, and a `Close` button for open conversations.
  - **Right column** — Selected-conversation thread.
    - Header: status pill + topic + conversation id (truncated) + a `Close` button.
    - Turn thread: alternating user-message bubbles (left-aligned, grey background) and agent-reply bubbles (right-aligned, accent background). User bubbles show the sanitized text with small PII chip(s) above if categories were found. Compaction markers appear as a horizontal rule with a grey `compacted — N turns summarised` label and an expand toggle to show the summary text.
    - Message input at the bottom: a text input and a `Send` button (disabled while status is `AGENT_THINKING`).
- Status pill colours: OPEN=green, AGENT_THINKING=yellow, COMPACTING=blue, CLOSED=muted, FAILED=red.
- PII chip colours per category: email=red, phone=orange, ssn=red, payment-card-number=red, address=yellow, person-name=orange.
- The raw message text is never displayed on this screen — only the sanitized form. The raw text is accessible via `GET /api/conversations/{id}` and reading each turn's entity record.
