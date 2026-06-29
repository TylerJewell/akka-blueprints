# UI mockup — guardrails-pattern

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Guardrails Pattern</title>`.

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
- Headline: `Guardrails<span class="accent"> Pattern</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded prompt or type your own. Select a policy profile.
  3. Click **Submit**.
  4. Watch the card progress: prompt-guard check → agent reply → reply-guard check.
- Card **How it works**: one paragraph on submit → prompt-guard → agent → reply-guard; one paragraph on the two guardrail hooks (before-llm-call blocks bad inputs; before-agent-response rejects bad outputs).
- Card **Components**: rows per component (InteractionEntity, InteractionWorkflow, AssistantAgent, PromptGuardrail, ReplyGuardrail, InteractionView, InteractionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 2 Guardrails as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `prompt_screened_before_llm: true` declaration in Data is filled and prominent. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, G2). ID badges coloured: G1 and G2 both red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a prompt. <span class="accent">See the guardrails work.</span>`. Subtitle: `One agent, two guardrail hooks.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Policy profile` (GENERAL / CONSERVATIVE / STRICT), `Prompt` textarea (with a "Load seeded example" link that fills the prompt text), `Submitted by` text input, and a yellow `Submit` button.
    - Live list below: one card per interaction, newest-first. Each card shows status pill, policy-profile badge, age. BLOCKED cards show the matched `ruleId` in red.
  - **Right column** — Selected-interaction detail.
    - Header: status pill + policy-profile badge + prompt text (truncated if long).
    - Input guardrail row: a green PASSED chip or red BLOCKED chip with the `ruleId` and `reason`.
    - Reply section (visible only when status is REPLY_RECORDED): `category` chip, `confidence` bar (0–100%), reply `text` in a readable block, `citations` list.
    - Output guardrail row: a green PASSED chip or yellow REJECTED chip with the `reason` (if visible — only shown on final state; retry outcomes are internal).
    - Guardrail timeline at the bottom: a compact two-row table showing input-guard result (timestamp, passed/blocked) and output-guard result (timestamp, passed/rejected on final attempt).
- Status pill colours: SUBMITTED=muted, PROMPT_CHECKED=blue, BLOCKED=red, REPLYING=yellow, REPLY_RECORDED=green, FAILED=red.
- Policy-profile badge colours: GENERAL=muted, CONSERVATIVE=blue, STRICT=orange.
- Category chip colours: FACTUAL=blue, CREATIVE=purple, EXTRACTION=teal, REFUSAL=yellow, OTHER=muted.
