# UI mockup — triage-router-ui

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Triage Agent (UX)</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Triage Agent <span class="accent">(UX)</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export, the key was handled during `/akka:specify` per Lesson 25), then four numbered steps (open App UI tab, wait 30 s for the simulator to drop the first message, click any message card to inspect, optionally click Unblock on a `BLOCKED` message).
- Card **How it works**: one paragraph on the route → handoff → guardrail → publish flow; one paragraph on the single governance mechanism (before-agent-response guardrail).
- Card **Components**: rows per component (simulator, queue, router agent, billing handler, product handler, draft guardrail, workflow, entity, view, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent typed, 2 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `decisions.authority_level = autonomous-with-guardrail`, `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`, and `data.pii_handled_by_sanitizer_before_llm = false` (this baseline does not include a sanitizer — a deployer would add one for production use). Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- One row: G1 (guardrail · yellow badge). Row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the routing. <span class="accent">Catch the violations.</span>`. Subtitle: `Simulated messages drop every 30 s. The router picks a handler; the guardrail catches out-of-policy drafts.`
- Layout: three-column.
  - **Left column** — Live message list, sorted newest-first. Each card shows:
    - Header: status pill, category chip (BILLING yellow, PRODUCT cyan, UNCLEAR muted), message age.
    - Original subject line.
    - Channel chip (email / chat / web-form).
  - **Centre column** — Selected message: original subject + body (read-only), the routing block (category badge + confidence + reason).
  - **Right column** — The chosen handler's draft:
    - Handler tag (billing yellow, product cyan).
    - Draft subject + body.
    - Action chip (`REFUND_INITIATED`, `INFO_PROVIDED`, etc.).
    - Guardrail block: green check + "allowed" when verdict allowed; red badge + violations list when blocked.
    - When `status = BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/messages/{id}/unblock`.
    - When `status = RESOLVED`: the published response carries a green "Published" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — no handler invoked" block with the `escalationReason`.
- Status pill colours: `RECEIVED` muted, `ROUTED_BILLING` yellow, `ROUTED_PRODUCT` cyan, `REPLY_DRAFTED` purple, `BLOCKED` red, `RESOLVED` green, `ESCALATED` amber.
