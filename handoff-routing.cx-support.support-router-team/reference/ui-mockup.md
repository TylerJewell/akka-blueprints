# UI mockup — support-multi-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Support Multi-Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Support <span class="accent">Multi-Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block, then four numbered steps (open App UI, wait 30 s for the simulator to drop the first case, click any case to inspect, optionally click Unblock on a `BLOCKED` case).
- Card **How it works**: one paragraph on the sanitize → route → specialist-handoff → guardrail → publish flow; one paragraph on the two governance mechanisms.
- Card **Components**: rows per component (simulator, queue, sanitizer, router agent, billing specialist, technical specialist, account specialist, draft guardrail, workflow, entity, view, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent typed, 3 AutonomousAgents, 1 Guardrail Agent, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs with answers from `risk-survey.yaml`. The distinctive declarations are `data.pii = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `decisions.authority_level = autonomous-with-guardrail`, and `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`. Jurisdictional and operational `TO_BE_COMPLETED_BY_DEPLOYER` values render muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Two rows: G1 (guardrail · yellow badge), S1 (sanitizer · cyan badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the routing. <span class="accent">Catch the violations.</span>`. Subtitle: `Simulated cases drop every 30 s. The router picks the specialist; the guardrail catches the misses.`
- Layout: three-column.
  - **Left column** — Live case list, sorted newest-first. Each card shows:
    - Header: status pill, category chip (BILLING yellow, TECHNICAL cyan, ACCOUNT purple, UNCLEAR muted), case age.
    - Sanitized subject (the user never sees the raw subject).
    - PII categories found (small muted chips).
  - **Centre column** — Selected case: redacted subject + body (read-only), the routing block (category badge + confidence + reason).
  - **Right column** — The chosen specialist's draft:
    - Specialist tag (billing yellow, technical cyan, account purple).
    - Draft subject + body.
    - Action chip (`REFUND_INITIATED`, `INFO_PROVIDED`, `ACCESS_RESTORED`, etc.).
    - Guardrail block: green check + "allowed" when verdict allowed; red badge + violations list when blocked.
    - When `status = BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/cases/{id}/unblock`.
    - When `status = RESOLVED`: the published response carries a green "Published" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — no specialist invoked" block in place of the draft, with the `escalationReason`.
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `ROUTED_BILLING` yellow, `ROUTED_TECHNICAL` cyan, `ROUTED_ACCOUNT` purple, `RESOLUTION_DRAFTED` blue, `BLOCKED` red, `RESOLVED` green, `ESCALATED` amber.
