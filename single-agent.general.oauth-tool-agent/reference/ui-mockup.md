# UI mockup — ae-oauth

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: OAuth Tool Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `OAuth <span class="accent">Tool Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a token (`full-access`, `read-only`, or `calendar-only`) from the dropdown.
  3. Type a request (or load a seeded example) and click **Run**.
  4. Watch the session card transition through TOKEN_RESOLVED → RUNNING → COMPLETED, with per-tool disposition chips in the detail pane.
- Card **How it works**: one paragraph on request → token resolve → agent → scope-guardrail per call → result; one paragraph on the before-tool-call guardrail and how denied calls are recorded rather than silently dropped.
- Card **Components**: rows per component (SessionEntity, SessionWorkflow, ToolCallerAgent, OAuthScopeGuardrail, TokenRegistry, ToolRegistry, SimulatedToolExecutor, SessionView, SessionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type (plain class for registry/executor/guardrail).
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, 1 Guardrail, 2 Registries, 1 Executor).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `decisions.authority_level = automated` and `oversight.human_in_loop = false` are the distinctive answers — enforcement is in-process with no human gate. `data.pii = false` is prominent (token ids are opaque, no user PII flows through the agent). Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Pick a token. <span class="accent">Run a request.</span>`. Subtitle: `One agent, one guardrail, per-call scope enforcement.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Token` (`full-access` / `read-only` / `calendar-only` / custom), `Request` textarea (with a "Load example" link that fills the field with a seeded multi-tool request), `Submitted by` text input, and a yellow `Run` button.
    - Scope preview below the dropdown: when a seeded token is selected, a row of scope chips appears immediately showing the selected token's scope set (e.g., `calendar:read`, `calendar:write`, `files:read`). Custom token input shows a placeholder row.
    - Live list below: one card per session, newest-first. Each card shows status pill, outcome badge (when result landed), session request truncated to 60 chars, age.
  - **Right column** — Selected-session detail.
    - Header: status pill + outcome badge + request text (full).
    - Resolved scope chips: a row of coloured chips for each scope in `token.scopes`.
    - Tool calls table: columns tool name, required scope, disposition chip (ALLOWED=green / DENIED=red), output or denial reason (truncated with expand).
    - Agent summary paragraph at the bottom.
- Status pill colours: PENDING=muted, TOKEN_RESOLVED=blue, RUNNING=yellow, COMPLETED=green, FAILED=red.
- Outcome badge colours: SUCCESS=green, PARTIAL=yellow, DENIED=red.
- Disposition chip colours: ALLOWED=green, DENIED=red.
