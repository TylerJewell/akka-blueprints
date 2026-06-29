# UI mockup — per-tenant-agent-fleet

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Per-Customer Success CRM Agents</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Per-Customer Success <span class="accent">CRM Agents</span>`. **No subtitle.**
- Card **Try it**: a `/akka:build` (Claude Code) block, then three numbered steps (register a customer, watch the agent onboard them, send a chat message and see the reply). No env-var export block.
- Card **How it works**: one paragraph naming the control-plane fleet coordinator, the per-customer research and welcome pipeline, and the ongoing-chat capability over persistent memory.
- Card **Components**: table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, event-sourced = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, customer state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (3 agents, 1 workflow, 2 event-sourced entities, 2 views).
- Four mermaid cards (component graph, sequence, customer state machine, ER) with the Akka theme variables AND the Lesson 24 state-label CSS overrides (white state names, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`). Without these the state names render black-on-black and the arrow labels clip.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sections in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each question block is rendered with the question text and the chips / textareas / list widgets in their selected state per `risk-survey.yaml`.
- Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer"); unanswered blocks get `opacity: 0.45`.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges carry a colored mechanism pill: sanitizer = teal, guardrail = red, hotl = muted, eval-periodic = blue. The four controls are S1 (sanitizer · pii), G1 (guardrail · before-tool-call), HO1 (hotl · deployer-runtime-monitoring), E1 (eval-periodic · performance-monitor).
- Rows expand vertically on click to show rationale + implementation; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Register a customer. <span class="accent">Watch the agent go to work.</span>` Subtitle: `The simulator also drips a customer every 60 s so the board is never empty.`
- Form card: a "Customer name" field, an "Email" field, an optional "CRM tier" selector (Standard / Premium / Enterprise), a `Register` button (yellow).
- The fleet board: a list of customer cards, one per registered customer. Each card shows:
  - the customer name, email, tier chip, and status chip (status colours: registered muted, onboarding yellow, ready green);
  - the profile summary once research completes;
  - the welcome status line: "Welcome sent" (green) or "Welcome blocked: <reason>" (red);
  - the latest eval score as a small score chip;
  - a **Chat panel** — a message input and Send button that posts to `POST /api/customers/{id}/chat`; each reply appears inline below the input; the conversation history scrolls. New turns arrive live via `/api/customers/{id}/chat/sse`.
- The fleet-health panel: a separate section below the fleet board, titled "Fleet Health". Shows the latest `FleetHealthSummary` list from `GET /api/fleet/health`. Each row shows the customer name, days-since-last-interaction, and recommendation. Refreshes on load and every 5 minutes.
- The fleet board updates live — status changes over `/api/customers/sse` — with no page reload.
