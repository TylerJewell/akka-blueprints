# UI mockup — it-helpdesk

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: IT Help Desk</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `IT Help <span class="accent">Desk</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block, then four numbered steps (open App UI, wait 30 s for the simulator to drop the first request, click any request card to inspect, optionally click Unblock on a `TICKET_BLOCKED` request).
- Card **How it works**: one paragraph on the sanitize → classify → handoff → guardrail → file ticket → publish flow; one paragraph on the two governance mechanisms.
- Card **Components**: rows per component (simulator, queue, sanitizer, classifier agent, access specialist, infra specialist, software specialist, routing judge, ticket guardrail, workflow, entity, view, scorer, endpoints) with Kind column coloured by primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 3 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the Lesson 24 CSS overrides so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `data.credentials = true`, `data.secrets_handled_by_sanitizer_before_llm = true`, `decisions.ticket_creation_gated_by_guardrail = true`, and `oversight.reviewer_must_unblock_guardrail_failures = true`. Deployer-specific fields render muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Two rows: G1 (guardrail · yellow badge), S1 (sanitizer · cyan badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the routing. <span class="accent">Catch the policy violations.</span>`. Subtitle: `Simulated requests drop every 30 s. The classifier picks the specialist; the ticket guardrail catches the policy misses.`
- Layout: three-column.
  - **Left column** — Live request list, sorted newest-first. Each card shows:
    - Header: status pill, category chip (ACCESS amber, INFRASTRUCTURE cyan, SOFTWARE green, UNCLEAR muted), request age, routing score chip (1–5; colour-graded: 1–2 red, 3 amber, 4–5 green).
    - Sanitized subject (never the raw subject).
    - Secret categories detected (small muted chips).
  - **Centre column** — Selected request: redacted subject + body (read-only), the classification block (category badge + confidence + reason), and the routing score block (number + rationale + scoredAt).
  - **Right column** — The chosen specialist's resolution:
    - Specialist tag (access amber, infra cyan, software green).
    - Response subject + body.
    - Action chip (`TICKET_FILED`, `SELF_SERVICE_RESOLVED`, `RUNBOOK_LINKED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`).
    - If `proposedTicket` present: a filed-ticket card (title, assignee group, priority badge).
    - Guardrail block: green check + "allowed" when verdict passed; red badge + violations list when blocked.
    - When `status = TICKET_BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/requests/{id}/unblock`.
    - When `status = RESOLVED`: the published response carries a green "Published" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — no specialist invoked" block with the `escalationReason`.
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `CLASSIFIED` blue, `ROUTED_ACCESS` amber, `ROUTED_INFRASTRUCTURE` cyan, `ROUTED_SOFTWARE` green, `RESOLUTION_DRAFTED` purple, `TICKET_BLOCKED` red, `RESOLVED` green, `ESCALATED` amber.

The routing score chip is the most distinctive surface — a continuous per-request quality signal visible in the list and broken down in the centre column detail view. Priority badges on the proposed-ticket card (red for P1, amber for P2, muted for P3) give the technician an at-a-glance view of queue urgency.
