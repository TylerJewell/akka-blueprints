# UI mockup — 311-triage

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: 311 Constituent Triage Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in a prior iteration must be deleted from the HTML; `display:none` is not sufficient. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `311 Constituent <span class="accent">Triage Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export, the key was handled during `/akka:specify` per Lesson 25), then four numbered steps (open App UI, wait 30 s for the simulator to drop the first request, click any request to inspect, optionally click Review on a `FLAGGED_FOR_REVIEW` request).
- Card **How it works**: one paragraph on the sanitize → triage → route-guardrail → handoff → publish flow; one paragraph on the two governance mechanisms plus the eval.
- Card **Components**: rows per component (simulator, queue, sanitizer, triage agent, route guardrail, public-works specialist, permits specialist, triage judge, workflow, entity, view, scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 2 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip at the top.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `data.pii = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `decisions.authority_level = autonomous-with-guardrail`, `purpose.sector = public-sector`, and `oversight.reviewer_must_resolve_flagged_routes = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Three rows: S1 (sanitizer · cyan badge), G1 (guardrail · yellow badge), E1 (eval-event · blue badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the triage. <span class="accent">Catch the misroutes.</span>`. Subtitle: `Simulated requests drop every 30 s. Triage picks the department; the route guardrail catches ambiguous calls before any specialist is invoked.`
- Layout: three-column.
  - **Left column** — Live request list, sorted newest-first. Each card shows:
    - Header: status pill, category chip (`PUBLIC_WORKS` amber, `PERMITS_ZONING` cyan, `UNCLEAR` muted), request age, triage score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Sanitized subject (the constituent's raw subject with PII tokens replaced).
    - PII categories found (small muted chips: name, phone, email, address).
  - **Centre column** — Selected request: redacted subject + description (read-only), the triage block (category badge + confidence + reason), the route verdict block (green check "approved" or red badge + flags list), and the triage score block (number + rationale + scoredAt).
  - **Right column** — The chosen specialist's response:
    - Department tag (public-works amber, permits-zoning cyan).
    - Response subject + body.
    - Action chip (`WORK_ORDER_CREATED`, `PERMIT_INFO_PROVIDED`, `REFERRAL_ISSUED`, etc.).
    - When `status = RESOLVED`: the published response carries a green "Published" stamp with `finishedAt`.
    - When `status = FLAGGED_FOR_REVIEW`: a yellow flag block showing the route flags list and a **Review** button. Clicking Review opens a small textarea for a note and two buttons: "Approve route" (posts `decision="approve"`) and "Escalate" (posts `decision="escalate"`).
    - When `status = ESCALATED`: a muted "Escalated — no specialist invoked" block with the `escalationReason`.
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `TRIAGED` blue, `ROUTE_APPROVED` teal, `ROUTED_PUBLIC_WORKS` amber, `ROUTED_PERMITS_ZONING` cyan, `RESPONSE_DRAFTED` purple, `FLAGGED_FOR_REVIEW` yellow, `RESOLVED` green, `ESCALATED` red.

The route verdict block is the most distinctive surface — it makes the before-agent-invocation guardrail tangible: a green check means the route passed inspection before the specialist was called; a flagged block means no specialist was ever invoked.
