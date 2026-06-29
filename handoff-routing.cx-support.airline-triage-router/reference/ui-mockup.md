# UI mockup — airline-triage-router

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Custom Orchestration Airline Assistant</title>`.

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
- Headline: `Custom Orchestration Airline <span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block, then four numbered steps (open App UI, wait 30 s for the feeder to drop the first request, click any request card to inspect, optionally click Unblock on a `BLOCKED` request).
- Card **How it works**: one paragraph on the sanitize → classify → handoff → guardrail → publish flow; one paragraph on the four governance mechanisms (PNR sanitizer, before-tool-call guardrail, before-agent-response guardrail, on-decision eval).
- Card **Components**: rows per component (feeder, queue, sanitizer, intent router, booking specialist, change specialist, baggage specialist, status specialist, routing judge, tool-call guardrail, response guardrail, workflow, entity, view, eval scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents typed — router + judge + 2 guardrails = 4 typed Agents, 4 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs with answers from `risk-survey.yaml`. Distinctive declarations: `data.pii = true`, `data.booking-reference = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `decisions.authority_level = autonomous-with-guardrail`, `decisions.destructive_tool_calls_gate_by_guardrail = true`, `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`. Jurisdictional and operational fields rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Four rows: S1 (sanitizer · cyan badge), G1 (guardrail · yellow badge — before-tool-call), G2 (guardrail · yellow badge — before-agent-response), E1 (eval-event · blue badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the routing. <span class="accent">Catch the violations.</span>`. Subtitle: `Simulated passenger requests drop every 30 s. The classifier picks the specialist; two guardrails catch the misses.`
- Layout: three-column.
  - **Left column** — Live request list, sorted newest-first. Each card shows:
    - Header: status pill, intent chip (`BOOKING` yellow, `CHANGE` orange, `BAGGAGE` purple, `STATUS` cyan, `UNCLEAR` muted), request age, routing score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Sanitized subject (raw subject never shown).
    - PII categories found (small muted chips).
  - **Centre column** — Selected request: redacted subject + body (read-only), the classification block (intent badge + confidence + reason), and the routing score block (number + rationale + scoredAt).
  - **Right column** — The chosen specialist's draft:
    - Specialist tag (`booking` yellow, `change` orange, `baggage` purple, `status` cyan).
    - Draft subject + body.
    - Outcome chip (`BOOKING_CONFIRMED`, `CHANGE_APPLIED`, `BAGGAGE_CLAIM_FILED`, `STATUS_PROVIDED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`).
    - Tool-call guardrail block: shown only when a destructive action was attempted. Green check + "permitted" when allowed; red badge + `denialReason` when denied.
    - Response guardrail block: green check + "allowed" when verdict allowed; red badge + violations list when blocked.
    - When `status = BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/requests/{id}/unblock`.
    - When `status = RESOLVED`: the published response carries a green "Published" stamp with `finishedAt`.
    - When `status = UNRESOLVED`: a muted "Unresolved — no specialist invoked" block with the `unresolvedReason` (typically "Intent UNCLEAR").
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `CLASSIFIED` blue, `ROUTED_BOOKING` yellow, `ROUTED_CHANGE` orange, `ROUTED_BAGGAGE` purple, `ROUTED_STATUS` cyan, `RESOLUTION_DRAFTED` indigo, `BLOCKED` red, `RESOLVED` green, `UNRESOLVED` amber.

The routing score chip and the two-layer guardrail surface are the distinguishing UI elements — the score gives a continuous per-request signal on classifier quality, and the tool-call verdict block makes the pre-execution governance step visible to the operator.
