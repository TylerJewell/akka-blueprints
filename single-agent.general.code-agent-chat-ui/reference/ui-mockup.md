# UI mockup — code-agent-chat-ui

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Gradio UI Code Agent</title>`.

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
- Headline: `Gradio UI <span class="accent">Code Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **New session** or pick one of the seeded conversation starters.
  3. Type a question and click **Send**.
  4. Watch the agent's tool steps and plan-revision badges appear, then the final response stream in.
- Card **How it works**: one paragraph on message → agent → tool calls → plan revision → response; one paragraph on the two guardrails (before-tool-call, before-agent-response).
- Card **Components**: rows per component (ChatSessionEntity, ToolCallValidator, ChatSessionWorkflow, CodeAssistantAgent, ToolCallPolicy, ResponseGuardrail, ChatView, ChatEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 ToolCallPolicy + 1 ResponseGuardrail as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `code-execution-environment-isolated: true` and `web-search-domain-allowlist-enforced: true` are the distinctive pre-filled answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, G2). ID badges coloured: G1 red (before-tool-call guardrail), G2 red (before-agent-response guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Watch the agent work.</span>`. Subtitle: `One agent, two guardrails, full tool-call audit trail.`
- Layout: two-column.
  - **Left column** — Session panel.
    - **New session** button at the top. Below: a list of sessions ordered newest-first. Each item shows the session title (or "Untitled session" if derived title not yet set), a status pill, and the age.
    - Five seeded conversation starter chips below the new-session button: "What is the current Python version?", "Calculate the 20th Fibonacci number", "Summarize recent news about event sourcing", "Write a prime-checking function", "Top 3 results for 'event sourcing patterns'". Clicking a chip creates a new session and sends that message.
  - **Right column** — Chat thread.
    - Session header: title + status pill.
    - Message list, scrolled to bottom. Each message:
      - USER messages: right-aligned bubble, plain text.
      - ASSISTANT messages: left-aligned bubble, markdown-rendered content (code blocks in a monospace fence, bold/italic preserved). Below the content, a collapsible **Steps** panel listing each tool call: tool name chip, input summary, status badge (APPROVED / BLOCKED / COMPLETED / FAILED), and output summary. Plan-revision entries appear inline between tool-call rows as `[Replanned at step N] <revised goal>` in muted italic.
    - At the bottom: a textarea for the next message + **Send** button. Send is disabled while the session is in `ACTIVE` state (agent is running).
- Status pill colours: INITIALIZING=muted, ACTIVE=yellow (pulsing), IDLE=green, FAILED=red.
- Tool status badge colours: PENDING=muted, APPROVED=blue, BLOCKED=red, COMPLETED=green, FAILED=red.
- If a session is in FAILED state, the last assistant bubble shows a red border and the error reason below the content.
