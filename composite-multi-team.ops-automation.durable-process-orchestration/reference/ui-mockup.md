# UI mockup — sk-process

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: SK Process</title>`.

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
- Headline: `SK <span class="accent">Process</span>`. **No subtitle.**
- Card **Try it**: a `/akka:build` (Claude Code) block, then three numbered steps (submit a job template, watch the execution team run the steps, watch the summary finalise). No env-var export block.
- Card **How it works**: one paragraph naming the coordinator pipeline and the three desks' different coordination capabilities (planning delegates to the step planner, execution is a team over a shared board, validation is a moderated quality panel) plus the pause/resume capability.
- Card **Components**: table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, event-sourced = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, job state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (4 agents, 2 workflows, 3 event-sourced entities, 2 views).
- Four mermaid cards (component graph, sequence, job state machine, ER) with the Akka theme variables AND the Lesson 24 state-label CSS overrides (white state names, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`). Without these the state names render black-on-black and the arrow labels clip.
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
- ID badges carry a colored mechanism pill: guardrail = red, hotl = muted, halt = orange. The four controls are HO1 (hotl · deployer-runtime-monitoring), H1 (halt · graceful-degradation), G1 (guardrail · before-agent-response), G2 (guardrail · before-tool-call).
- Rows expand vertically on click to show rationale + implementation; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a job. <span class="accent">Watch the team run it.</span>` Subtitle: `The simulator also drips a template every 90 s so the board is never empty.`
- Form card: a "Process template" field (e.g., `dependency-audit`), an optional "Payload" field, an optional "Submitted by" field, and a `Submit` button (yellow).
- The job board: one column or card lane per `JobStatus` (Submitted · Planned · Executing · Paused · Validating · Approved · Completed), or a single list with a status chip per card. Each job card shows:
  - the template name and status chip (status colours: submitted muted, executing/validating yellow, paused amber, approved green, completed blue);
  - the plan objective once planned;
  - a step-progress line (e.g., "2 / 4 steps done") during execution, drawn from the step board;
  - **pause and resume buttons** visible when status is `EXECUTING` or `PAUSED` respectively — these post to `/api/jobs/{id}/pause` and `/api/jobs/{id}/resume`;
  - the quality verdict chip (PASS green / RETRY red) and any `mustRetry` step names;
  - the two step-signal scores (batch / validation) as small score chips;
  - once completed: the summary overview, the `resultRef` link, and an **audit-review box** — a small form (reviewed-by, an outcome toggle APPROVED/FLAGGED, a notes field, a Submit) that posts to `POST /api/jobs/{id}/audit-review`; once a review exists, the box shows the recorded outcome and notes instead.
- The step-board panel: grouped by `StepStatus` (Open · Claimed · Done). Each step card shows its name, the job it belongs to, the claiming executor (when claimed), and the output token count (when done). Cards move between groups live via SSE.
- Both the job board and the step board update live — the job board over `/api/jobs/sse`, the step board over `/api/steps/sse` — with no page reload.
