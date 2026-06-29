# UI mockup — mcp-chat-server

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Chat Server (MCP)</title>`.

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
- Headline: `Chat Server <span class="accent">(MCP)</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **New conversation** and give it a title, or load a seeded example.
  3. Type a message and click **Send**.
  4. Watch the turn card transition through RUNNING → REPLIED and read the agent's reply with tool-call audit chips.
- Card **How it works**: one paragraph on message → agent → MCP-tool-calls → guardrails → reply; one paragraph on the two governance mechanisms (before-tool-call guardrail, before-agent-response guardrail).
- Card **Components**: rows per component (ConversationEntity, ChatWorkflow, ChatAgent, ToolCallGuardrail, ReplyGuardrail, ConversationView, ChatEndpoint, AppEndpoint, MockMcpServer) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, 1 MockMcpServer, plus 2 Guardrails as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = direct-action` and `oversight.human_in_loop = false` declarations are filled and prominent. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, G2). ID badges coloured: G1 and G2 both red (guardrail). The mechanism sub-type distinguishes them: G1 shows `before-tool-call`, G2 shows `before-agent-response`.
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Read the reply.</span>`. Subtitle: `One agent, two guardrails, external tools.`
- Layout: two-column.
  - **Left column** — Conversation list + new-conversation panel.
    - **New conversation** button at the top. Clicking it reveals an inline form: `Title` text input, `Created by` text input, and a `Create` button. Seeded examples appear as quick-create links below the form.
    - Conversation list below: one card per conversation, newest-first. Each card shows title, status pill (`ACTIVE` / `CLOSED`), turn count, and last-active age. Clicking a card loads it into the right column.
  - **Right column** — Active conversation thread + message entry.
    - Thread: scrollable list of turns, newest-at-bottom. Each turn shows:
      - **User bubble** — message text, `sentBy`, timestamp.
      - **Agent bubble** (when REPLIED) — reply text, tool-call audit chips (one chip per `ToolCall`; `BLOCKED` chips are amber, `ALLOWED` chips are blue), iterations-used badge, timestamp.
      - **Turn status pill** — RECEIVED=muted, RUNNING=yellow (animated), REPLIED=green, FAILED=red.
    - Message entry at the bottom: `Message` textarea, `Sent by` text input, and a yellow **Send** button. Sending is disabled while the previous turn is in `RUNNING` state.
- Status pill colours: RECEIVED=muted, RUNNING=yellow, REPLIED=green, FAILED=red.
- Conversation status pill colours: ACTIVE=green, CLOSED=muted.
- Tool-call chip colours: ALLOWED=blue, BLOCKED=amber.
