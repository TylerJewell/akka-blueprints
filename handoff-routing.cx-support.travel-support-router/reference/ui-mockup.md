# UI mockup — travel-support-router

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Travel Support Router</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Travel Support <span class="accent">Router</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block, then four numbered steps (open App UI, wait 30 s for the simulator to drop the first request, click any request card to inspect, optionally click Confirm or Reject at the confirmation gate).
- Card **How it works**: one paragraph on the sanitize → route → specialize → guardrail → confirm → publish flow; one paragraph on the three governance mechanisms.
- Card **Components**: rows per component (simulator, queue, sanitizer, router agent, four specialists, routing judge, booking guardrail, confirmation gate, workflow, entity, view, scorer, endpoints) with Kind column coloured by primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 4 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24: `stateDiagram .label foreignObject { overflow: visible; }` and theme variable `transitionLabelColor: '#cccccc'` so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs styled per `governance.html` pattern, answers from `risk-survey.yaml`. Distinctive declarations: `data.pii = true`, `data.travel-document = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `decisions.authority_level = hitl-confirmed`, `decisions.booking_mutations_require_passenger_confirmation = true`, `oversight.human_in_loop = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Three rows: S1 (sanitizer · cyan badge), G1 (guardrail · yellow badge), H1 (hitl · purple badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the handoff. <span class="accent">Confirm the change.</span>`. Subtitle: `Simulated requests drop every 30 s. The router picks the specialist; the guardrail catches policy misses; the passenger has final say.`
- Layout: three-column.
  - **Left column** — Live request list, sorted newest-first. Each card shows:
    - Header: status pill, category chip (FLIGHTS blue, HOTELS amber, CAR_RENTAL teal, EXCURSIONS purple, UNCLEAR muted), request age, routing score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Redacted subject (the user never sees the raw subject).
    - PII categories found (small muted chips, e.g. `passport-number`, `booking-reference`).
  - **Centre column** — Selected request: redacted subject + body (read-only), the routing block (category badge + confidence + reason), and the routing score block (number + rationale + scoredAt).
  - **Right column** — The chosen specialist's proposed resolution:
    - Specialist tag chip (FLIGHTS blue, HOTELS amber, CAR_RENTAL teal, EXCURSIONS purple).
    - Proposed subject + body.
    - Action chip (`BOOKING_CHANGED`, `BOOKING_CANCELLED`, `INFO_PROVIDED`, etc.).
    - Guardrail block: green check + "allowed" when verdict allowed; red badge + violations list when blocked.
    - Confirmation block (when `status = AWAITING_CONFIRMATION`): the plain-language summary, a deadline countdown, and two buttons — **Confirm** (green) and **Reject** (muted red). Clicking either posts to `/api/requests/{id}/confirm`.
    - When `status = PASSENGER_REJECTED`: a muted "Passenger rejected — no booking mutation applied." block.
    - When `status = BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/requests/{id}/unblock`.
    - When `status = RESOLVED`: the published response carries a green "Published" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — no specialist invoked" block with the `escalationReason`.
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `ROUTED` blue, `ROUTED_FLIGHTS` blue, `ROUTED_HOTELS` amber, `ROUTED_CAR_RENTAL` teal, `ROUTED_EXCURSIONS` purple, `GUARDRAIL_CLEARED` green, `AWAITING_CONFIRMATION` yellow-pulse, `RESOLUTION_DRAFTED` indigo, `BLOCKED` red, `RESOLVED` green, `PASSENGER_REJECTED` rose, `ESCALATED` amber.

The confirmation block at `AWAITING_CONFIRMATION` is the most distinctive surface — it makes the HITL mechanism tangible: the workflow is visibly paused, the passenger's plain-language summary is front and centre, and both action buttons are present until the deadline.
