# UI mockup — spot-edge-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Spot Agent (Edge)</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Spot Agent <span class="accent">(Edge)</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded route (indoor patrol / outdoor inspection / payload drop) or define a custom waypoint list.
  3. Click **Submit mission**.
  4. Watch the mission progress through PLANNING → EXECUTING → COMPLETED (or see the approval gate and safety halt in action).
- Card **How it works**: one paragraph on submit → plan → (approval gate) → execute → complete; one paragraph on the three governance mechanisms (motion-command guardrail, automatic safety halt, human approval gate).
- Card **Components**: rows per component (MissionEntity, TelemetryMonitor, MissionWorkflow, SpotNavigatorAgent, CommandGuardrail, SimulatedSpotMcp, MissionView, MissionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Simulated MCP as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `sector = industrial-robotics` and `decisions_surface = autonomous-with-oversight` declarations are prominent. `location-fine-grained = true` (robot GPS) is flagged. Many fields carry `TO_BE_COMPLETED_BY_DEPLOYER` and render in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, A1). ID badges coloured: G1 red (guardrail), H1 orange (halt), A1 yellow (hitl).
- Each row is click-to-expand; the expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a mission. <span class="accent">Watch it execute.</span>`. Subtitle: `One agent, three physical-safety mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Route` (Indoor Patrol / Outdoor Inspection / Payload Drop / custom), `Mission name` text input, `Payload instructions` textarea (with a "Load seeded example" link that fills all fields), `Operator callsign` text input, and a yellow `Submit mission` button.
    - Live list below: one card per mission, newest-first. Each card shows status pill, outcome badge (when completed), mission name, and age.
  - **Right column** — Selected-mission detail.
    - Header: status pill + outcome badge + mission name.
    - Telemetry tiles (4 tiles in a row): battery %, joint load bar, obstacle proximity, E-stop indicator — update live via SSE.
    - Waypoint route: numbered list of waypoints with terrain-type chip and a reached/pending/skipped indicator per waypoint.
    - Approval panel (visible only when status = PENDING_APPROVAL): the plain-English command summary, a table of proposed commands (command name, params, rationale), and **Approve** / **Reject** buttons. Approve button is yellow; Reject is muted.
    - Outcome section (visible when status = COMPLETED or PARTIAL or ABORTED): outcome status badge, waypointsReached count / total, distanceTravelledM, anomaliesObserved list.
    - Halt reason section (visible when status = HALTED): cause chip, sensor snapshot at halt time (battery %, joint load, proximity), last command string.
- Status pill colours: SUBMITTED=muted, PLANNING=blue, PENDING_APPROVAL=yellow, EXECUTING=yellow, COMPLETED=green, HALTED=red, FAILED=red.
- Outcome badge colours: COMPLETED=green, PARTIAL=yellow, ABORTED=muted, HALTED=red.
- Telemetry danger thresholds highlighted: battery below 15 % turns the tile red; joint load above 0.9 turns the tile red; obstacle proximity below 0.3 m turns the tile red.
