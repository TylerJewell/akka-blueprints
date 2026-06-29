# UI mockup — realtime-conversational-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Realtime Conversational Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Realtime <span class="accent">Conversational Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Enter a customer ID (or use the seeded demo ID) and click **Start session**.
  3. Type a message and click **Send** to converse with the agent.
  4. Click **End session** to close and see the session summary and quality score.
- Card **How it works**: one paragraph on session-open → greet → converse loop → summarize; one paragraph on the `before-agent-response` guardrail and the `TurnSummarizer`.
- Card **Components**: rows per component (SessionEntity, SessionWorkflow, ConversationalAgent, ResponseGuardrail, TurnSummarizer, SessionView, SessionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 1 Summarizer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = fully-autonomous` declaration is filled and prominent — the agent's replies reach the customer without a human read-gate. `oversight.human_on_loop = true` and `overview.human_in_loop = false` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Start a session. <span class="accent">Have a conversation.</span>`. Subtitle: `One agent, one guardrail on every turn.`
- Layout: two-column.
  - **Left column** — Start panel + session list.
    - Start panel: `Customer ID` text input (pre-filled with demo ID `customer-demo-01`), `Agent persona` dropdown (retail-support / account-management / returns-and-refunds / general-support), and a teal **Start session** button.
    - Session list below: one card per session, newest-first. Each card shows status pill, turn count badge, customer ID, agent persona label, and age. Closed sessions show their quality score chip.
  - **Right column** — Selected-session conversation thread.
    - Header: status pill + quality score chip (when closed) + `customerId` + persona label.
    - Conversation thread: alternating customer / agent message bubbles, newest at the bottom. Customer bubbles on the left; agent bubbles on the right. Each agent bubble shows a small guardrail-triggered badge (yellow `⚠ guardrail`) if `guardrailTriggered=true`. Turns in `WAITING_FOR_AGENT` show a pulsing typing indicator on the right side.
    - Message input at the bottom: `Customer message` textarea + **Send** button. Input is disabled when the session is in `WAITING_FOR_AGENT` or `SUMMARIZING` or `CLOSED` state.
    - **End session** button at the top-right of the right pane, enabled only while session is `ACTIVE`.
    - Session summary section at the bottom of the right pane (visible only when `status = CLOSED`): quality score stars (1–5), summary text paragraph, total turns count, and guardrail events count.
- Status pill colours: GREETING=blue, ACTIVE=green, WAITING_FOR_AGENT=yellow, SUMMARIZING=blue, CLOSED=muted, FAILED=red.
- Quality score chip colours: score 4–5=green, score 3=yellow, score 1–2=red. Score ≤ 2 highlights the session card border red.
