# UI mockup — guardrails-demo

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Agent with Guardrails</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Agent with <span class="accent">Guardrails</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a demo scenario from the dropdown (allowed-topic, blocked-topic, content-policy retry) or type a custom message.
  3. Click **Send**.
  4. Watch the turn card transition through CHECKING_INPUT → GENERATING → COMPLETED (or BLOCKED).
- Card **How it works**: one paragraph on message receipt → topic check → agent call → content check → reply; one paragraph on the two guardrail hooks (before-llm-call and before-agent-response) and what each intercepts.
- Card **Components**: rows per component (SessionEntity, SessionWorkflow, ConversationAgent, TopicPolicyGuardrail, ContentPolicyGuardrail, SessionView, SessionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 2 Guardrails as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `input_filtered_before_llm: true` declaration in Data is filled and prominent. `decisions.authority_level = informational-only` and `oversight.human_in_loop: false` (this is a fully automated response loop) are the distinctive answers. Most other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, G2). ID badges both red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Watch the guardrails work.</span>`. Subtitle: `One agent, two guardrail hooks — input and output.`
- Layout: two-column.
  - **Left column** — Session panel + session list.
    - Session panel: dropdown `Demo scenario` (Allowed topic / Blocked topic / Content-policy retry / Custom), `Message` textarea (pre-filled from scenario), a `New session` link to clear context, and a yellow `Send` button.
    - Session list below: one card per session, newest-first. Each card shows status badge, userId, age, and turn count.
  - **Right column** — Selected session's turn timeline.
    - Session header: sessionId, userId, status badge, opened-at timestamp.
    - Turn timeline: one card per turn, chronological.
      - Each turn card shows:
        - User message text (grey background).
        - Topic-check badge: ALLOWED (green) or BLOCKED (red + matched topic name).
        - If BLOCKED: a note "LLM not called — topic blocked." No reply section.
        - If ALLOWED: content-policy iteration badge (e.g. "1 iteration" green or "2 retries" yellow).
        - Agent reply text (white background).
        - Turn status chip and timestamps.
- Status turn chip colours: RECEIVED=muted, CHECKING_INPUT=blue, BLOCKED=red, GENERATING=yellow, COMPLETED=green, FAILED=red.
- Topic-check badge: ALLOWED=green, BLOCKED=red.
- Content-policy iteration badge: 1 iteration=green, 2 iterations=yellow, 3 iterations=orange.

The user message is always visible in the turn card regardless of outcome. This is intentional: blocked messages are retained in the session log for audit; the UI surfaces them so a reviewer can see what was blocked and why.
