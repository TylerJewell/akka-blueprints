# UI mockup — guardrails-side-by-side

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: GuardrailsSideBySide</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Guardrails <span class="accent">Side by Side</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded prompt (governance question / off-topic / policy-violating-response) or type your own.
  3. Click **Submit**.
  4. Watch the card transition through SCREENING → AGENT_RUNNING → VALIDATING → RESPONDED (or BLOCKED / RESPONSE_BLOCKED).
- Card **How it works**: one paragraph describing the two-guardrail pipeline (input screener gates the prompt; main policy agent answers; output validator gates the reply); one paragraph on the eval step.
- Card **Components**: rows per component (PromptEntity, GuardrailWorkflow, InputScreenerAgent, PolicyAgent, OutputValidatorAgent, ResponseEvaluator, PromptView, PromptEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 AutonomousAgents — 1 primary + 2 guardrail, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, 1 Evaluator as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the two distinctive pre-filled answers. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, G2, E1). ID badges coloured: G1 red (guardrail / input), G2 orange (guardrail / output), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a prompt. <span class="accent">Watch both guardrails fire.</span>`. Subtitle: `One advisory agent, gated by an input screener and an output validator.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Policy profile` (standard / strict / permissive), `Prompt` textarea (with three "Load seeded example" links: governance question, off-topic, policy-violating), `Submitted by` text input, and a yellow `Submit` button.
    - Live list below: one card per prompt, newest-first. Each card shows status pill, decision chip (when terminal), eval score chip (when available), prompt text truncated to 60 chars, age.
  - **Right column** — Selected-prompt detail.
    - Header: status pill + decision chip + eval score chip + prompt text.
    - **Input screening section**: screener verdict badge (PASS green / BLOCK red) + screener reason.
    - **Agent response section**: shown only when `agentResponse` is present. Displays the reply when `validationVerdict.result == PASS`; shows `"[Response blocked — see Output Validation section]"` when `status == RESPONSE_BLOCKED`.
    - **Output validation section**: validator verdict badge (PASS green / BLOCK orange) + validator reason + triggered rule (if any).
    - **Eval section** (shown only when `eval` is present): a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: RECEIVED=muted, SCREENING=yellow, AGENT_RUNNING=yellow, VALIDATING=yellow, RESPONDED=green, BLOCKED=red, RESPONSE_BLOCKED=orange, FAILED=red.
- Decision chip colours: RESPONDED=green, BLOCKED=red, RESPONSE_BLOCKED=orange.
- Eval score chip: 1–2=red, 3=yellow, 4–5=green.

The raw agent reply is never displayed when `status == RESPONSE_BLOCKED`. The UI shows the validator's block reason in the output-validation section. A reviewer who needs the raw reply for audit can fetch `/api/prompts/{id}` and read `agentResponse.reply` from the JSON — the entity always stores it for audit even when it was blocked from the caller.
