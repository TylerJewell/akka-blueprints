# UI mockup — routing-classifier-pattern

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Routing Workflow</title>`.

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
- Headline: `Routing <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block, then four numbered steps (open App UI, wait 30 s for the simulator to drop the first message, click any message to inspect, optionally click Unblock on a `REPLY_BLOCKED` message).
- Card **How it works**: one paragraph on the classify → validate-route → handoff → screen → publish flow; one paragraph on the two governance mechanisms (before-invocation route guardrail and before-response reply guardrail).
- Card **Components**: rows per component (simulator, queue, ingestor, routing classifier, route guardrail, general agent, refund agent, technical agent, reply guardrail, workflow, entity, view, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (4 Agents typed, 3 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. Distinctive declarations: `decisions.routing_decisions_validated_by_guardrail = true`, `decisions.authority_level = autonomous-with-guardrail`, `oversight.human_in_loop = false`, `oversight.reviewer_must_unblock_reply_failures = true`, `oversight.route_blocked_messages_require_operator_action = false`. `pii_handled_by_sanitizer_before_llm = false` is rendered with a yellow caution chip since incoming messages are not sanitized in this baseline. Deployer-only fields are rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Two rows: G1 (guardrail · yellow badge, hook: before-agent-invocation), G2 (guardrail · yellow badge, hook: before-agent-response). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the routing. <span class="accent">Catch the violations.</span>`. Subtitle: `Messages drop every 30 s. The classifier picks the specialist; two guardrails protect the output.`
- Layout: three-column.
  - **Left column** — Live message list, sorted newest-first. Each card shows:
    - Header: status pill, route chip (`GENERAL` muted-blue, `REFUND` yellow, `TECHNICAL` cyan, `UNROUTABLE` muted), message age.
    - Message subject (raw; no sanitizer in this baseline).
    - Channel chip (`email` / `chat` / `web-form` / `phone-transcript`).
  - **Centre column** — Selected message: subject + body (read-only), route decision block (route badge + confidence + reason), route verdict block (green "approved" or red "rejected" + rejectionReason).
  - **Right column** — The chosen specialist's draft reply:
    - Specialist tag (`general` muted-blue, `refund` yellow, `technical` cyan).
    - Reply subject + body.
    - Action chip (`REFUND_INITIATED`, `INFO_PROVIDED`, etc.).
    - Reply guardrail block: green check + "allowed" when verdict allowed; red badge + violations list when blocked.
    - When `status = REPLY_BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/messages/{id}/unblock`.
    - When `status = PUBLISHED`: the published reply carries a green "Published" stamp with `finishedAt`.
    - When `status = ROUTE_BLOCKED`: a red "Route blocked" notice in place of the specialist block, showing the `rejectionReason`. No unblock path.
    - When `status = ABANDONED`: a muted "Abandoned — unroutable" block in place of the specialist block, showing the `abandonReason`.
- Status pill colours: `RECEIVED` muted, `CLASSIFIED` blue, `ROUTE_BLOCKED` red, `ROUTED_GENERAL` muted-blue, `ROUTED_REFUND` yellow, `ROUTED_TECHNICAL` cyan, `REPLY_DRAFTED` purple, `REPLY_BLOCKED` red, `PUBLISHED` green, `ABANDONED` amber.

The two-position governance model is the most distinctive surface of this baseline: the centre column shows both the route guardrail verdict (step 2) and the reply guardrail verdict (step 4), making the defence-in-depth visible at a glance.
