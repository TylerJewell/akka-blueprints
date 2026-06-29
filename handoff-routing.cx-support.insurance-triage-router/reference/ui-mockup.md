# UI mockup — auto-insurance-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Auto Insurance Agent</title>`.

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
- Headline: `Auto Insurance <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export, the key was handled during `/akka:specify` per Lesson 25), then four numbered steps (open App UI, wait 30 s for the simulator to drop the first request, click any request to inspect, optionally click Unblock on a `BLOCKED` request).
- Card **How it works**: one paragraph on the sanitize → triage → four-way handoff → guardrail → publish flow; one paragraph on the two governance mechanisms.
- Card **Components**: rows per component (simulator, queue, sanitizer, triage agent, claims specialist, policy specialist, rewards specialist, roadside specialist, triage judge, response guardrail, workflow, entity, view, scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 4 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip at the top.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `data.pii = true`, `data.vehicle-identification-number = true`, `data.drivers-license-number = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `decisions.authority_level = autonomous-with-guardrail`, `decisions.claim_settlement_decisions_made_by_ai = false`, and `oversight.reviewer_must_unblock_guardrail_failures = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Two rows: S1 (sanitizer · cyan badge), G1 (guardrail · yellow badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the handoff. <span class="accent">Catch the violations.</span>`. Subtitle: `Simulated member requests drop every 30 s. Triage picks the specialist; the guardrail catches the misses.`
- Layout: three-column.
  - **Left column** — Live request list, sorted newest-first. Each card shows:
    - Header: status pill, category chip (CLAIM yellow, POLICY blue, REWARDS purple, ROADSIDE cyan, UNCLEAR muted), request age, triage score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Sanitized subject (the user never sees the raw subject).
    - PII categories found (small muted chips: member-id, policy-number, vin, drivers-license, phone, account-number).
  - **Centre column** — Selected request: redacted subject + body (read-only), the triage block (category badge + confidence + reason), and the triage score block (number + rationale + scoredAt).
  - **Right column** — The chosen specialist's draft:
    - Specialist tag (claims yellow, policy blue, rewards purple, roadside cyan).
    - Draft subject + body.
    - Action chip (`CLAIM_ACKNOWLEDGED`, `ROADSIDE_DISPATCHED`, `REWARDS_REDEEMED`, etc.).
    - Confirmation reference chip (shown only when `confirmationRef` is non-null — roadside dispatches).
    - Guardrail block: green check + "allowed" when verdict allowed; red badge + violations list when blocked.
    - When `status = BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/requests/{id}/unblock`.
    - When `status = RESOLVED`: the published response carries a green "Published" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — no specialist invoked" block in place of the draft, with the `escalationReason`.
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `TRIAGED` blue, `ROUTED_CLAIM` yellow, `ROUTED_POLICY` blue, `ROUTED_REWARDS` purple, `ROUTED_ROADSIDE` cyan, `RESPONSE_DRAFTED` orange, `BLOCKED` red, `RESOLVED` green, `ESCALATED` amber.

The confirmation reference chip is the most distinctive surface for roadside requests — the `RSA-` reference generated by `RoadsideSpecialist` appears in the right column as soon as the specialist returns, giving the member an immediate handle on their dispatch.
