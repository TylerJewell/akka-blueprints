# UI mockup — triage-handoff-routing

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SK Multi-Agent Handoff</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The exemplar's canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `SK Multi-Agent <span class="accent">Handoff</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export, the key was handled during `/akka:specify` per Lesson 25), then four numbered steps (open App UI, wait 30 s for the simulator to drop the first turn, click any turn to inspect, optionally click Review on a blocked turn).
- Card **How it works**: one paragraph on the classify → routing-check → handoff → specialist → response-check → publish flow; one paragraph on the two guardrail mechanisms and where they fire.
- Card **Components**: rows per component (simulator, queue, intent classifier, triage agent, routing guardrail, account specialist, product specialist, return specialist, response guardrail, workflow, entity, view, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents typed [TriageAgent], 2 Agents typed [RoutingGuardrail + ResponseGuardrail], 3 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 1 TimedAction, 2 HttpEndpoints). Tile labels: `2 Typed Agents (classifier)`, `2 Typed Agents (guardrails)`, `3 AutonomousAgents`, `1 Workflow`, `2 ESEs`, `1 View`, `1 Consumer`, `1 TimedAction`, `2 HttpEndpoints`.
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `decisions.routing_validated_by_guardrail_before_specialist = true`, `decisions.authority_level = draft-only`, `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`, and `data.sensitive_data_flagged_by_classifier_before_llm = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Two rows: G1 (guardrail · yellow badge, before-agent-invocation), G2 (guardrail · yellow badge, before-agent-response). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the handoff. <span class="accent">Two guardrails, one reply.</span>`. Subtitle: `Simulated turns drop every 30 s. Triage picks the specialist; two guardrails protect the routing decision and the reply.`
- Layout: three-column.
  - **Left column** — Live turn list, sorted newest-first. Each card shows:
    - Header: status pill, intent chip (ACCOUNT cyan, PRODUCT blue, RETURNS amber, UNCLEAR muted), turn age, routing verdict chip (green check or red X).
    - Normalised text excerpt (first 80 chars).
    - Channel indicator (chat, email, web-widget).
  - **Centre column** — Selected turn: normalised text (read-only), the triage block (intent badge + confidence + reason), and the routing verdict block (green "allowed" check with reason, or red "blocked" with reason when `ROUTING_BLOCKED`).
  - **Right column** — The chosen specialist's draft:
    - Specialist tag (account cyan, product blue, returns amber).
    - Reply text.
    - Action chip (`INFORMATION_PROVIDED`, `ACCOUNT_UPDATED`, `RETURN_INITIATED`, etc.).
    - Response verdict block: green check + "allowed" when `allowed=true`; red badge + violations list when blocked.
    - When `status = ROUTING_BLOCKED` or `status = RESPONSE_BLOCKED`: a Review button (amber) opens a note textarea with a `publish` toggle and posts to `/api/turns/{id}/review`.
    - When `status = RESOLVED`: the published reply carries a green "Published" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — no reply sent" block with the `escalationReason`.
- Status pill colours: `RECEIVED` / `NORMALISED` muted, `TRIAGED` blue, `ROUTING_BLOCKED` red, `ROUTED_ACCOUNT` cyan, `ROUTED_PRODUCT` blue-purple, `ROUTED_RETURNS` amber, `REPLY_DRAFTED` purple, `RESPONSE_BLOCKED` red, `RESOLVED` green, `ESCALATED` orange.

The routing verdict chip is the most distinctive surface in this blueprint — it is the before-agent-invocation guardrail made tangible as a per-turn badge visible at a glance in the list column, before the user even opens the turn detail.
