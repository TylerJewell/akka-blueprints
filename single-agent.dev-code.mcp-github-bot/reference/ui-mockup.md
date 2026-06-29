# UI mockup — mcp-github-bot

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: MCP GitHub Bot</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. Canonical implementation:

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
- Headline: `MCP GitHub <span class="accent">Bot</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Enter your GitHub token and a `owner/repo` target, or use mock mode.
  3. Pick a seeded request (list repos, list issues, create issue, add comment) or type your own.
  4. Click **Send request** and watch the session move through SUBMITTED → RUNNING → COMPLETED.
- Card **How it works**: one paragraph on request → agent → GitHub MCP tool calls → response; one paragraph on the two governance controls (before-tool-call guardrail and operator halt flag).
- Card **Components**: rows per component (BotSessionEntity, HaltFlagEntity, BotSessionWorkflow, GitHubBotAgent, ToolCallGuardrail, BotSessionView, BotSessionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 2 EventSourcedEntities, 1 View, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = autonomous-action` declaration is prominent. `oversight.human_on_loop = true` and `operator_can_halt_writes = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 dark-red / crimson (halt).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a request. <span class="accent">GitHub responds.</span>`. Subtitle: `One agent, two governance controls around the tool surface.`
- Layout: two-column.
  - **Left column** — Controls + form + live list.
    - **Halt flag toggle** at the top: a prominent chip showing `WRITES ACTIVE` (green) or `WRITES HALTED` (red) with an **Enable halt** / **Disable halt** button next to it. Calls `POST /api/halt/enable` or `/disable`.
    - Request form: `Repository` text input (placeholder `owner/repo`), `GitHub token` password input (placeholder `ghp_...`; never stored), `Request` textarea (with a "Load seeded example" dropdown offering the 4 seed requests), `Submitted by` text input, and a yellow `Send request` button.
    - Live list below: one card per session, newest-first. Each card shows status pill, session age, and a one-line truncation of the user request.
  - **Right column** — Selected-session detail.
    - Header: status pill + sessionId + age.
    - User request block: the original natural-language request.
    - Tool-call log: an ordered list. Each row shows: tool name chip (coloured by tool class — read=blue, write=orange), `outcome` chip (`success`=green, `error`=yellow, `blocked`=red), input summary (monospace, truncated), result summary.
    - Blocked-calls section (only shown if `blockedCallCount > 0`): a red-bordered box listing each blocked call with its rejection reason.
    - Agent response: the `agentMessage` paragraph.
- Status pill colours: SUBMITTED=muted, RUNNING=yellow, COMPLETED=green, FAILED=red.
- Tool outcome chip colours: success=green, error=yellow, blocked=red.
- The `githubToken` value the user typed is never shown in the right pane or logged anywhere in the UI. Once submitted, the form's token field is cleared.
