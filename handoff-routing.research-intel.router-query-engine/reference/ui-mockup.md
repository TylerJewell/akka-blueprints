# UI mockup — router-query-engine

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Router Query Engine Workflow</title>`.

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
- Headline: `Router Query Engine <span class="accent">Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block, then four numbered steps (open App UI, wait 30 s for the simulator to drop the first question, click any query card to inspect, optionally submit a manual question via the form).
- Card **How it works**: one paragraph on the route → handoff → answer → publish flow; one paragraph on the on-decision eval mechanism.
- Card **Components**: rows per component (simulator, queue, consumer, router agent, structured-data engine, semantic-search engine, routing judge, workflow, entity, view, scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents typed, 2 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip at the top.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `data.pii = false`, `decisions.authority_level = draft-only`, and `oversight.human_in_loop = false`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- One row: E1 (eval-event · blue badge). The row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the router decide. <span class="accent">See the score.</span>`. Subtitle: `Simulated questions drop every 30 s. The router picks the engine; the judge scores the call.`
- Layout: two-column.
  - **Left column** — Live query list, sorted newest-first. Each card shows:
    - Header: status pill, engine-type chip (STRUCTURED yellow, SEMANTIC cyan, UNCLEAR muted), query age, routing score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Question text (truncated to 80 chars).
    - Source chip ("simulator" or "manual").
  - **Right column** — Selected query detail:
    - Full question text (read-only).
    - Routing decision block: engine-type badge + confidence + reason.
    - Routing score block: number + rationale + scoredAt.
    - Engine answer block:
      - Engine tag chip (structured yellow, semantic cyan).
      - Answer text.
      - Action chip (`DIRECT_LOOKUP`, `AGGREGATION_RESULT`, `CONCEPT_SUMMARY`, `SYNTHESIS`, `ESCALATED`).
      - Source references list (small muted chips, one per `sourceRef`).
      - When `status = ANSWERED`: a green "Published" stamp with `finishedAt`.
      - When `status = ESCALATED`: a muted "Escalated — no engine invoked" block with the `escalationReason`.
- Status pill colours: `RECEIVED` muted, `ROUTING` blue, `ROUTED_STRUCTURED` yellow, `ROUTED_SEMANTIC` cyan, `ANSWERING` purple, `ANSWERED` green, `ESCALATED` amber.
- Manual question form at the bottom of the left column: a textarea for `questionText` and a Submit button. On submit, `POST /api/queries` is called; the new query card appears at the top of the list within one SSE tick.

The routing score chip is the most distinctive surface — a continuous, per-query number visible at-a-glance in the list and broken down in the right column. It is the on-decision eval mechanism made tangible.
