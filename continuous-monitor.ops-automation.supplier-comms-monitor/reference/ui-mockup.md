# UI mockup — supplier-comms-monitor

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Supplier Communications Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Supplier Communications <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the simulator to drop a PO, review the outreach draft, click Approve / Escalate).
- Card **How it works**: one paragraph on the poll → assess → draft → buyer-approve flow; one paragraph on the three governance mechanisms.
- Card **Components**: rows per component (poller, feed queue, consumer, risk agent, outreach agent, workflow, entity, view, eval runner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent, 1 AutonomousAgent, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.
- Mermaid CSS overrides from Lesson 24 MUST be present (state label colour, `foreignObject overflow:visible`, `transitionLabelColor #cccccc`).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `commercial-confidential: true` declaration in Data is the most distinctive chip. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, E1). ID badges coloured: G1 yellow (guardrail), H1 pink (hitl), E1 green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Monitor the PO board. <span class="accent">Approve every outreach.</span>`. Subtitle: `Simulated POs drop every 20 s. Drafts for material changes are held until you act.`
- Layout: two-column.
  - **Left column** — Live PO board, sorted newest-first. Each card shows:
    - Header: status pill, risk tier chip, PO age.
    - Item description and supplier name.
    - Risk factors as small muted chips.
  - **Right column** — the selected PO detail.
    - Feed event fields (PO id, supplier, item, qty, promised date).
    - Risk assessment rationale (read-only).
    - Outreach draft (textarea — editable before approval).
    - Approve button (yellow). Escalate button (border). Escalate opens a small "Escalation Note" textarea.
    - After action, the accuracy score chip (when populated).
- Status pill colours: OPENED=muted, RISK_ASSESSED=blue, OUTREACH_DRAFTED=blue, AWAITING_BUYER_APPROVAL=yellow, OUTREACH_SENT=green, PROCUREMENT_REVIEW=red, ON_TRACK_CONFIRMED=green, CLOSED=muted.
- Risk tier chip colours: ON_TRACK=green, AT_RISK=yellow, CRITICAL=red.

Editing the outreach body before Approve is allowed — the approved body is what is logged as the sent outreach.
