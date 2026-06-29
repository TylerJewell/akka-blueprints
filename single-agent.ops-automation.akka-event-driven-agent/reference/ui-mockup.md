# UI mockup — ambient-durable-agent-pubsub

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Ambient Durable Agent on Pub/Sub</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Ambient Durable Agent on <span class="accent">Pub/Sub</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded request payload (tropical-storm-alert / cold-front-advisory / heatwave-forecast / standard-daily-request) or paste your own JSON.
  3. Click **Publish**.
  4. Watch the card transition through RECEIVED → VALIDATED → SANITIZED → FORECASTING → COMPLETED.
- Card **How it works**: one paragraph on topic-message arrival → validate → sanitize → agent forecast → record result; one paragraph on the two governance mechanisms (before-agent-invocation guardrail, PII sanitizer).
- Card **Components**: rows per component (RequestEntity, TopicConsumer, WeatherWorkflow, WeatherAgent, PayloadGuardrail, RequestSanitizer, RequestView, RequestEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Sanitizer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with Lesson 24 CSS overrides and `themeVariables` block (nodeTextColor, stateLabelColor, transitionLabelColor `#cccccc`; state-label foreignObject `overflow:visible`). Without these, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `oversight.human_in_loop = false` and `human_on_loop = true` are the distinctive answers for this ambient (non-interactive) pattern — flagged with a brief explanatory note. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered as muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Publish a message. <span class="accent">Watch it complete.</span>`. Subtitle: `One agent, two governance gates before it.`
- Layout: two-column.
  - **Left column** — Publish panel + live list.
    - Publish panel: dropdown `Seeded payload` (tropical-storm-alert / cold-front-advisory / heatwave-forecast / standard-daily-request / custom), `Location` text input, `Horizon` select (`HOURLY_6H` / `DAILY_3D` / `WEEKLY_7D`), `Alert categories` multi-chip input, `Requester contact` text input (labelled "(optional — will be sanitized)"), and a yellow `Publish` button.
    - "Load seeded example" link fills all fields from the selected seeded payload.
    - Live list below the panel: one card per request, newest-first. Each card shows status pill, hazard badge (when report landed), location, age.
  - **Right column** — Selected-request detail.
    - Header: status pill + hazard badge + location.
    - Validation result: green chip "VALID" or red chip with violation code list.
    - Sanitized payload preview: the PII category chips (`email`, `phone`, `name`, `host`) above a JSON block showing the sanitized location and horizon.
    - Weather report section (visible when status ≥ COMPLETED): conditionsSummary, temperature range bar, wind chip, precipitation chip, and hazard detail (red-bordered panel when `hazardFlag = true`).
    - Raw payload link at the bottom: "View audit record (GET /api/requests/{id})" — opens the full JSON in a new tab. This is how reviewers access `rawPayload.requesterContact` without the UI displaying it.
- Status pill colours: RECEIVED=muted, VALIDATED=blue, SANITIZED=teal, FORECASTING=yellow, COMPLETED=green, FAILED=red.
- Hazard badge: `HAZARD` in red when `hazardFlag = true`; absent otherwise.
- The raw `requesterContact` value is never displayed in the UI — only the redacted form appears in the sanitized preview. Reviewers who need the original fetch the API endpoint directly.
