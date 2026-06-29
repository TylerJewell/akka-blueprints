# UI mockup — akka-handoff-routing

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: OpenAI Agents SDK Handoff Routing</title>`.

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
- Headline: `OpenAI Agents SDK <span class="accent">Handoff Routing</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export), then four numbered steps (open App UI, wait 30 s for the first simulated turn, click any conversation to inspect, optionally click Unblock on a `BLOCKED` conversation).
- Card **How it works**: one paragraph on the filter → guardrail → triage → handoff → resolve → publish flow; one paragraph on the three governance mechanisms.
- Card **Components**: rows per component (simulator, queue, message filter, routing guardrail, triage agent, billing specialist, technical specialist, handoff judge, workflow, entity, view, scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 2 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `data.pii = true`, `data.pii_handled_by_filter_before_llm = true`, `decisions.authority_level = autonomous-with-guardrail`, and `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`. Jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Three rows: G1 (guardrail · yellow badge), S1 (sanitizer · cyan badge), E1 (eval-event · blue badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the handoff. <span class="accent">Catch the blocked contexts.</span>`. Subtitle: `Simulated turns drop every 30 s. The guardrail gates before routing; the filter strips PII before any LLM call.`
- Layout: three-column.
  - **Left column** — Live conversation list, sorted newest-first. Each row shows:
    - Header: status pill, routing chip (BILLING yellow, TECHNICAL cyan, UNCLEAR muted), conversation age, handoff score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Filtered message excerpt (the user never sees the raw message).
    - PII categories found (small muted chips).
  - **Centre column** — Selected conversation: filtered message + prior-turn summaries (read-only), the guardrail block (green "passed" or red "blocked" + violation list), the routing block (category badge + confidence + reason), and the handoff score block (number + rationale + scoredAt).
  - **Right column** — The chosen specialist's reply:
    - Specialist tag (billing yellow, technical cyan).
    - Reply body.
    - Action chip (`REFUND_INITIATED`, `INFO_PROVIDED`, etc.).
    - When `status = BLOCKED`: the guardrail violation list + an Unblock button (yellow) that opens a note textarea and posts to `/api/conversations/{id}/unblock`. No specialist reply is shown because the specialist was never invoked.
    - When `status = RESOLVED`: the published reply carries a green "Published" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — no specialist invoked" block with the `escalationReason`.
- Status pill colours: `RECEIVED` / `FILTERED` muted, `GUARDRAIL_PASSED` blue, `ROUTED_BILLING` yellow, `ROUTED_TECHNICAL` cyan, `REPLY_READY` purple, `BLOCKED` red, `RESOLVED` green, `ESCALATED` amber.

The distinction from the response-guardrail pattern is visible in the layout: when a conversation is `BLOCKED`, the right column shows only the guardrail violation — there is no specialist draft, because the specialist was never called.
