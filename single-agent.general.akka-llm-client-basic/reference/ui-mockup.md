# UI mockup — akka-llm-client-basic

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Conversation API LLM Client</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The exemplar's `static-resources/index.html` has the canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Conversation<span class="accent">API</span> LLM Client`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded prompt from the dropdown or type your own.
  3. Click **Send**.
  4. Watch the turn appear as `PENDING` then transition to `REPLIED` with the model's response.
- Card **How it works**: one paragraph on prompt submission → agent call → guardrail check → reply recorded; one paragraph on the single governance mechanism (before-agent-response guardrail).
- Card **Components**: rows per component (ConversationEntity, ConversationAgent, ReplyGuardrail, ConversationView, ConversationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = informational` and `oversight.human_in_loop = false` declarations are filled and prominent. Most `TO_BE_COMPLETED_BY_DEPLOYER` fields are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a prompt. <span class="accent">Read the reply.</span>`. Subtitle: `One agent, one guardrail.`
- Layout: two-column.
  - **Left column** — Input panel + turn history list.
    - Input panel: dropdown `Seeded prompts` (factual / code / summarise / creative / custom), `Prompt` textarea (auto-fills from dropdown selection), and a yellow `Send` button.
    - Session controls: a `New session` link that resets the session id and clears the turn history display.
    - Turn history below: one card per turn, newest-first. Each card shows status pill, prompt preview (truncated to 60 chars), age, and reply preview when available.
  - **Right column** — Selected-turn detail.
    - Header: status pill + turn id chip + age.
    - Prompt block: the full prompt text in a bordered panel.
    - Reply block: the full reply text with optional code block rendering.
    - Metadata row: input tokens, output tokens, latency in ms, guardrail outcome badge (PASSED / REJECTED-RETRY / FAILED-ALL-RETRIES).
    - Timestamp: `generatedAt` formatted as local time.
- Status pill colours: PENDING=yellow, REPLIED=green, FAILED=red.
- Guardrail outcome badge colours: PASSED=green, REJECTED-RETRY=yellow, FAILED-ALL-RETRIES=red.

The selected-turn detail pane is blank until the user clicks a turn card in the left list. On first load, the most recent turn is auto-selected if one exists.
