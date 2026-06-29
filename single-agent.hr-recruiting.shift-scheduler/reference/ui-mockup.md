# UI mockup — nexshift

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: NexShift</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Nex<span class="accent">Shift</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded shift set (Nursing / Technician / Coordinator) from the department dropdown, or load the seeded example.
  3. Click **Generate schedule**.
  4. Watch the run transition through BUILDING → SCHEDULING → DRAFT, then click **Confirm schedule**.
- Card **How it works**: one paragraph on submit → build-roster → schedule → confirm; one paragraph on the one governance mechanism (before-tool-call guardrail on every `AssignShift` call).
- Card **Components**: rows per component (ScheduleEntity, ScheduleWorkflow, NexShiftAgent, AssignmentGuardrail, ScheduleView, ScheduleEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = draft-for-human-approval` declaration is filled and prominent. `oversight.human_in_loop = true` and `working-time-directive-hours-cap-applied = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- Row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit shifts. <span class="accent">Get a schedule.</span>`. Subtitle: `One agent, one guardrail on every write.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Department` (Nursing / Technician / Coordinator / custom), `Week starting` date picker, a "Load seeded example" link that populates shifts and employees, `Submitted by` text input, and a yellow `Generate schedule` button.
    - Live list below: one card per scheduling run, newest-first. Each card shows status pill, filled/total chip (e.g., `5 / 5`), blocked count chip if non-zero, department label, age.
  - **Right column** — Selected-run detail.
    - Header: status pill + filled/total chip + department + week label.
    - Open shifts: a table with `shiftId`, role, start–end time, required qualifications, and the assigned employee (or an `UNASSIGNED` badge).
    - Assignment table: columns shift id, employee name, start–end, status chip (`ASSIGNED` green / `GUARDRAIL_BLOCKED` red / `UNASSIGNED` muted), and blocked reason when applicable.
    - Employee roster summary: a compact list with `employeeId`, name, hours-scheduled/cap, qualifications.
    - When status is `DRAFT`: a prominent yellow `Confirm schedule` button at the bottom of the right pane.
- Status pill colours: PENDING=muted, BUILDING=blue, SCHEDULING=yellow, DRAFT=blue, CONFIRMED=green, FAILED=red.
- Assignment status chip colours: ASSIGNED=green, GUARDRAIL_BLOCKED=red, UNASSIGNED=muted.
- The confirm button only renders when `status === 'DRAFT'`; it is absent in all other states.
