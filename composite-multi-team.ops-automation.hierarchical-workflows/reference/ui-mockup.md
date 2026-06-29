# UI mockup — hierarchical-workflow-automation

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Hierarchical Workflow Automation</title>`.

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
- Headline: `Hierarchical Workflow <span class="accent">Automation</span>`. **No subtitle.**
- Card **Try it**: a `/akka:build` (Claude Code) block, then three numbered steps (submit an operations request, watch the teams run, watch the report deliver). No env-var export block — `/akka:specify` handled the key during generation.
- Card **How it works**: one paragraph naming the orchestrator pipeline and the three teams' different coordination capabilities (discovery delegates, execution is a team over a shared task board, validation is a moderated panel).
- Card **Components**: table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, event-sourced = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, workflow state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (6 agents, 2 workflows, 3 event-sourced entities, 2 views).
- Four mermaid cards (component graph, sequence, workflow state machine, ER) with the Akka theme variables AND the Lesson 24 state-label CSS overrides (white state names, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`). Without these the state names render black-on-black and the arrow labels clip.
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
- ID badges carry a colored mechanism pill: guardrail = red, eval-event = blue, hotl = muted. The four controls are G1 (guardrail · before-agent-response), G2 (guardrail · before-tool-call), E1 (eval-event · on-decision-eval), HO1 (hotl · live-compliance-review).
- Rows expand vertically on click to show rationale + implementation; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a request. <span class="accent">Watch the teams run it.</span>` Subtitle: `The simulator also drips a request every 60 s so the board is never empty.`
- Form card: an "Operations request" field, an optional "Requested by" field, a `Submit` button (yellow).
- The request board: one column or card lane per `WorkflowStatus` (Submitted · Planned · Discovering · Discovered · Executing · Executed · Validating · Validated · Completed), or a single list with a status chip per card. Each request card shows:
  - the description and status chip (status colours: submitted muted, executing/validating yellow, validated green, completed blue);
  - the plan objective once planned;
  - the discovery summary overview once discovered;
  - a task-progress line (e.g., "2 / 3 tasks done") during execution, drawn from the task board;
  - the validation verdict chip (PASS green / RETRY red) and any `mustRetry` titles;
  - the three audit-eval scores (discovery / execution / validation) as small score chips;
  - once completed: the report title, the report URL, and a **compliance-review box** — a small form (reviewed-by, an outcome toggle CLEARED/FLAGGED, a comments field, a Submit) that posts to `POST /api/requests/{id}/compliance-review`; once a review exists, the box shows the recorded outcome and comments instead.
- The task-board panel: grouped by `TaskStatus` (Open · Claimed · Done). Each task card shows its title, the request it belongs to, the claiming executor (when claimed), and the steps completed (when done). Cards move between groups live via SSE.
- Both the request board and the task board update live — the request board over `/api/requests/sse`, the task board over `/api/tasks/sse` — with no page reload.
