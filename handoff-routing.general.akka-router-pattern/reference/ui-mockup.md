# UI mockup — akka-router-pattern

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: OpenAI Agents Routing Pattern</title>`.

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
- Headline: `OpenAI Agents <span class="accent">Routing Pattern</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block, then four numbered steps (open App UI, wait 30 s for the simulator to drop the first request, click any request to inspect, optionally click Unblock on a `BLOCKED` request).
- Card **How it works**: one paragraph on the classify → guardrail → route → execute → publish flow; one paragraph on the two governance mechanisms.
- Card **Components**: rows per component (simulator, queue, queue consumer, classifier agent, routing guardrail, content specialist, code specialist, data specialist, routing judge, workflow, entity, view, scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 3 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs with answers from `risk-survey.yaml`. The distinctive declarations are `decisions.authority_level = autonomous-with-guardrail`, `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`, and `failure.failure_modes` covering prompt-injection and cross-domain confusion. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Two rows: G1 (guardrail · yellow badge), E1 (eval-event · blue badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the routing. <span class="accent">Catch the injections.</span>`. Subtitle: `Simulated requests drop every 30 s. The classifier picks the specialist; the guardrail blocks unsafe payloads.`
- Layout: three-column.
  - **Left column** — Live request list, sorted newest-first. Each card shows:
    - Header: status pill, domain chip (CONTENT amber, CODE cyan, DATA purple, UNKNOWN muted), request age, routing score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Request title.
    - Channel badge (`api`, `web-form`, `cli`).
  - **Centre column** — Selected request: title + body (read-only), the classification block (domain badge + confidence + reason), and the routing score block (number + rationale + scoredAt).
  - **Right column** — The chosen specialist's result:
    - Specialist tag (content amber, code cyan, data purple).
    - Result title + body.
    - Guardrail block: green check + "allowed" when verdict ALLOWED; red badge + violations list when denied.
    - When `status = BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/requests/{id}/unblock`.
    - When `status = COMPLETED`: the published result carries a green "Completed" stamp with `finishedAt`.
    - When `status = UNROUTABLE`: a muted "Unroutable — no specialist invoked" block with the `unroutableReason`.
- Status pill colours: `RECEIVED` muted, `CLASSIFIED` blue, `GUARDRAIL_PASSED` teal, `ROUTED_CONTENT` amber, `ROUTED_CODE` cyan, `ROUTED_DATA` purple, `EXECUTING` indigo, `BLOCKED` red, `COMPLETED` green, `UNROUTABLE` orange.

The routing score chip is the most distinctive surface — a continuous, per-request number visible at-a-glance in the list and broken down in the centre column. It is the on-decision eval mechanism made tangible.
