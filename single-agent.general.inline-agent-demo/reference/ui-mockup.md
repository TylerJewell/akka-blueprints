# UI mockup — inline-agent-demo

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Inline Agent</title>`.

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
- Headline: `Inline <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded agent definition (Q&A assistant / keyword extractor / tone classifier) and the matching example question, or supply your own.
  3. Click **Run**.
  4. Watch the card transition through RUNNING → COMPLETED.
- Card **How it works**: one paragraph on how the agent definition is assembled at call time (system prompt + output schema + allowed tools); one paragraph on the run lifecycle (validate → run → record).
- Card **Components**: rows per component (RunEntity, RunWorkflow, InlineAgentRunner, RunView, RunEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. Most fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic — this is intentional for a demo baseline that has no specific deployment context.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- The table body is empty — no controls are wired for this baseline. A text note explains: "This baseline ships with no controls. Fork the blueprint and add controls appropriate to your deployment context."

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Define an agent at runtime. <span class="accent">Run it.</span>`. Subtitle: `One agent, defined inline.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel:
      - Dropdown `Agent definition` (Q&A assistant / Keyword extractor / Tone classifier / custom). Selecting a seeded option fills the Agent definition textarea and the Question field.
      - `Agent definition` textarea — JSON object with `agentName`, `systemPrompt`, `outputSchema`, `allowedTools`. Editable.
      - `Question` textarea — the question to send to the agent.
      - `Submitted by` text input.
      - A yellow `Run` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, agent name, age.
  - **Right column** — Selected-run detail.
    - Header: status pill + agent name + run age.
    - Agent definition summary: `agentName`, a truncated `systemPrompt` (first 120 chars), and the `allowedTools` list (or "No tools" if empty).
    - Question: the submitted question in a read-only block.
    - Answer: the `response.answer` field in a monospace read-only block. If the answer is valid JSON, render it with syntax colouring.
    - Metadata row: `tokenCount` + `answeredAt`.
    - If status is `FAILED`, show `failureReason` in a red alert box instead of the answer block.
- Status pill colours: RECEIVED=muted, RUNNING=yellow, COMPLETED=green, FAILED=red.
