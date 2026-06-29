# UI mockup — smart-home-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Connected House Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Connected <span class="accent">House Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded command or type your own.
  3. Click **Send command**.
  4. Watch the card transition — for light commands: straight to DISPATCHED; for lock/thermostat: pause at AWAITING_CONFIRMATION, then confirm.
- Card **How it works**: one paragraph on receive → guard → confirm (if sensitive) → dispatch; one paragraph on the two governance mechanisms (before-tool-call guardrail, HITL confirmation).
- Card **Components**: rows per component (CommandEntity, DeviceRegistry, CommandWorkflow, HomeControlAgent, ActionGuardrail, CommandView, CommandEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: 1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 1 View, 2 HttpEndpoints, 1 Guardrail.
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. `decisions.authority_level = actuation` and `oversight.human_in_loop = true` are the distinctive answers. `data_classes.location-fine-grained = true` is highlighted (room-level presence inferred from device state). Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are rendered as muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 orange (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Issue a command. <span class="accent">Watch it happen.</span>`. Subtitle: `One agent, guardrail and resident confirmation around it.`
- Layout: two-column.
  - **Left column** — Command panel + device grid + live list.
    - Command panel: a text input `Your command` (with placeholder "Lock the front door"), a `Submitted by` text input, a dropdown `Seeded commands` (4 options), and a yellow `Send command` button.
    - Device grid below: one card per device showing displayName, location, deviceClass icon, and current state (on/off badge, brightness %, temperature set-point, lock icon). State updates in real time via SSE.
    - Live command list below: one card per command, newest-first. Each card shows status pill, device name + action badge (when action landed), age, and a rejection reason chip (for BLOCKED commands).
  - **Right column** — Selected-command detail.
    - Header: status pill + commandId truncated.
    - Raw command: monospace block.
    - Guard result: green "PASSED" badge or red "BLOCKED" badge with the `rejectionReason` text.
    - Proposed action (when guard passed): device display name, action type badge, numeric param if present, rationale sentence.
    - Confirmation panel (when status is AWAITING_CONFIRMATION): a yellow info banner "Resident confirmation required" showing the proposed action, and two buttons — green **Confirm** and grey **Cancel**.
    - Dispatch status: green "DISPATCHED" badge with the dispatched action, or grey "CANCELLED".
- Status pill colours: RECEIVED=muted, GUARD_PASSED=blue, BLOCKED=red, AWAITING_CONFIRMATION=yellow, DISPATCHED=green, CANCELLED=grey, FAILED=red.
- Action type badge colours: TURN_ON/TURN_OFF=blue, SET_BRIGHTNESS=purple, SET_TEMPERATURE=orange, LOCK/UNLOCK=red, OPEN/CLOSE=teal.
