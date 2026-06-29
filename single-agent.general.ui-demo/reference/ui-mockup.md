# UI mockup — ui-demo

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: AG-UI + CopilotKit Integration</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `AG-UI + <span class="accent">CopilotKit</span> Integration`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded prompt from the dropdown, or type your own question.
  3. Click **Send**.
  4. Watch the answer stream in, citation markers appear, and the suggested follow-up chip arrive.
- Card **How it works**: one paragraph on prompt submission → workflow start → agent call → guardrail → commit → SSE stream close; one paragraph on the `before-agent-response` guardrail and what it checks.
- Card **Components**: rows per component (SessionEntity, SessionWorkflow, CopilotAgent, ResponseGuardrail, SessionView, CopilotEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: TO_BE_COMPLETED_BY_DEPLOYER` field is shown prominently as an open deployer decision. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = false` are the distinctive pre-filled answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are rendered in muted italic to visually separate what the blueprint pre-fills from what the deployer must decide.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Read the answer.</span>`. Subtitle: `One agent, one guardrail at the response boundary.`
- Layout: two-column.
  - **Left column** — Session rail + new session button.
    - A `+ New session` button at the top mints a session with a default title and switches the right pane to it.
    - Session list: one card per session, newest-first. Each card shows session title, status pill, turn count, and age.
    - Clicking a session card loads its full history in the right pane.
  - **Right column** — Active session chat pane.
    - Chat history: alternating user bubble (right-aligned, muted background) and agent bubble (left-aligned, accent border). Each agent bubble contains the answer text with inline `[n]` citation markers rendered as small superscript chips. Below each agent bubble: a collapsible Citations list (index + source + snippet) and a `Suggested follow-up` chip that pre-fills the input when clicked.
    - Prompt input area at the bottom: a `Prompt` dropdown (5 seeded prompts) or free-text textarea, and a `Send` button. The button is disabled while a turn is `STREAMING`.
    - Streaming indicator: while status is `STREAMING`, the agent bubble shows a pulsing ellipsis at the end of the partial text.
- Status pill colours: `ACTIVE`=yellow (streaming/pending), `IDLE`=green (ready), `FAILED`=red.
- Turn status pill colours: `SUBMITTED`=muted, `STREAMING`=yellow (pulsing), `TURN_COMPLETE`=green, `TURN_FAILED`=red.
- Citation chips: small inline badges rendering `[n]` in the accent colour, hoverable to show source + snippet tooltip.
