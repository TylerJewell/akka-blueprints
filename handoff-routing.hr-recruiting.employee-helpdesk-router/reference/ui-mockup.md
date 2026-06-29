# UI mockup — employee-helpdesk-router

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Employee Helpdesk Router</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in a prior iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Employee Helpdesk <span class="accent">Router</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block, then four numbered steps (open App UI, wait 30 s for the simulator to drop the first question, click any question card to inspect, optionally click Unblock on a `BLOCKED` question).
- Card **How it works**: one paragraph on the sanitize → route → specialist handoff → guardrail → publish flow; one paragraph on the two governance mechanisms.
- Card **Components**: rows per component (simulator, queue, sanitizer, topic router, HR specialist, IT specialist, policy specialist, route judge, answer guardrail, workflow, entity, view, scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 3 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip at the top.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. Distinctive declarations: `data.pii = true`, `data.employee-id = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `decisions.authority_level = autonomous-with-guardrail`, `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Two rows: S1 (sanitizer · cyan badge), G1 (guardrail · yellow badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the routing. <span class="accent">Catch the violations.</span>`. Subtitle: `Simulated questions drop every 30 s. The router picks the specialist; the guardrail catches the misses.`
- Layout: three-column.
  - **Left column** — Live question list, sorted newest-first. Each card shows:
    - Header: status pill, topic chip (HR green, IT cyan, POLICY blue, UNCLEAR muted), question age, route score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Sanitized subject (the employee never sees the raw subject).
    - PII categories found (small muted chips).
  - **Centre column** — Selected question: redacted subject + body (read-only), the routing block (topic badge + confidence + reason), and the route score block (number + rationale + scoredAt).
  - **Right column** — The chosen specialist's draft:
    - Specialist tag (HR green, IT cyan, POLICY blue).
    - Draft subject + body.
    - Action chip (`INFO_PROVIDED`, `POLICY_CITED`, `TICKET_OPENED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`).
    - Guardrail block: green check + "allowed" when verdict allowed; red badge + violations list when blocked.
    - When `status = BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/questions/{id}/unblock`.
    - When `status = ANSWERED`: the published answer carries a green "Published" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — no specialist invoked" block in place of the draft, with the `escalationReason`.
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `ROUTED` blue, `ROUTED_HR` green, `ROUTED_IT` cyan, `ROUTED_POLICY` blue, `ANSWER_DRAFTED` purple, `BLOCKED` red, `ANSWERED` green, `ESCALATED` amber.

The route score chip is the distinctive surface — a continuous, per-question routing-quality number visible at-a-glance in the list and broken down in the centre column.
