# UI mockup — customer-service-tool-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: CXAgent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `CX<span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded customer from the dropdown and choose one of the four seeded scenarios (order status, refund, inventory check, account update).
  3. Click **Send**.
  4. Watch the session transition through ACTIVE → REPLIED (or ESCALATED if the refund exceeds the threshold).
- Card **How it works**: one paragraph on message → agent tool loop → reply → screener; one paragraph on the three governance mechanisms (before-tool-call guardrail, before-agent-response guardrail, PII log sanitizer).
- Card **Components**: rows per component (ConversationEntity, ReplyScreener, ConversationWorkflow, SupportAgent, ToolCallGuardrail, ReplyGuardrail, ConversationView, SupportEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, 2 Guardrails, 4 Tool classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` and `payment-card-data: true` declarations in Data are filled and prominent. `decisions.authority_level = automated-action` and `oversight.human_on_loop = true` (for escalated sessions) are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, G2). ID badges coloured: S1 green (sanitizer), G1 red (guardrail — tool-call), G2 red (guardrail — reply).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Get a grounded reply.</span>`. Subtitle: `One agent, two guardrails, one sanitizer.`
- Layout: two-column.
  - **Left column** — Composition panel + live session list.
    - Composition panel: dropdown `Customer` (CUST-1001 / CUST-1042 / CUST-2087), dropdown `Scenario` (Order status / Refund request / Inventory check / Account update / Custom), message textarea (pre-filled from scenario selection), and a yellow `Send` button. After the first message in a session, subsequent sends use the same session id.
    - Live session list below: one card per conversation session, newest-first. Each card shows status pill, customer id, last message preview (truncated 60 chars), age. ESCALATED sessions show an orange badge with the escalation ticket id.
  - **Right column** — Selected-session detail.
    - Header: status pill + customer id + `openedAt` age.
    - Conversation thread: each turn rendered as:
      - Customer bubble: message text + timestamp.
      - Tool-calls summary: a compact table (tool name, outcome badge ALLOWED/BLOCKED, truncated raw result). BLOCKED rows show the `blockReason` in muted italic.
      - Agent reply bubble: screened reply text + timestamp. If `piiCategoriesFound` is non-empty, a row of PII-category chips appears above the reply text.
    - At the bottom: a text input + Send button for follow-up messages in the same session.
- Status pill colours: OPEN=muted, ACTIVE=yellow, REPLIED=green, ESCALATED=orange, RESOLVED=blue, FAILED=red.
- Tool-call outcome badge: ALLOWED=green chip, BLOCKED=red chip.
- No raw PII values are ever shown in the UI — the screened form from `lastScreened.redactedReplyText` is used for the agent reply display.
