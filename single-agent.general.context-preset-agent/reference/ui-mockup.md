# UI mockup — context-preset-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: ContextPresetAgent</title>`.

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

And the DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Context<span class="accent">PresetAgent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick an environment (`dev` / `staging` / `prod`) and a role (`admin` / `guest`).
  3. Type a request and click **Submit**.
  4. Watch the card transition through PRESET_RESOLVED → EXECUTING → COMPLETED; check the tool call log.
- Card **How it works**: one paragraph on submit → resolve-preset → execute → audit; one paragraph on the two governance mechanisms (before-tool-call guardrail, CI configuration gate).
- Card **Components**: rows per component (PresetRequestEntity, PresetRegistry, PresetRequestWorkflow, ContextPresetAgent, ToolGatingGuardrail, PresetRequestView, PresetRequestEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 KeyValueEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `execute-with-constraints` authority level and `human_in_loop: false` declarations are the distinctive answers. `oversight.human_on_loop` and most compliance fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, C1). ID badges coloured: G1 red (guardrail), C1 yellow (ci-gate).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Pick a preset. <span class="accent">Submit a request.</span>`. Subtitle: `One agent, role-based tool gating.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: `Environment` dropdown (`dev` / `staging` / `prod`), `Role` dropdown (`admin` / `guest`), `Request` textarea, `Submitted by` text input, and a yellow `Submit` button. A small **Resolved preset preview** chip below the dropdowns updates immediately when either dropdown changes — showing `presetId`, `modelId`, and a comma-separated `allowedTools` list, so the caller knows what they'll get before submitting.
    - Live list below: one card per request, newest-first. Each card shows status pill, preset badge (`prod:admin`, `dev:guest`, etc.), blockedTools count chip (if > 0), and age.
  - **Right column** — Selected-request detail.
    - Header: status pill + preset badge + `requestText` truncated to 80 chars.
    - Resolved preset section: a collapsed card showing `modelId`, `allowedTools` list, and `instructionAddendum` excerpt.
    - Agent answer: the `answerText` paragraph.
    - Tool call log: a table with columns tool name, status chip (INVOKED=green / BLOCKED=red), input summary, output summary, timestamp.
    - Audit summary at bottom: total calls, blocked calls, `auditedAt`.
- Status pill colours: SUBMITTED=muted, PRESET_RESOLVED=blue, EXECUTING=yellow, COMPLETED=green, FAILED=red.
- Preset badge colours: prod=orange, staging=blue, dev=muted.
- Role indicator within preset badge: admin=bold, guest=normal weight.
