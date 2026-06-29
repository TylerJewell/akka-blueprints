# UI mockup — bug-assistant

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: BugAssistant</title>`.

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
- Headline: `Bug<span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded bug (Java NPE / Go deadlock / Python async race) or type your own, then click **Submit bug**.
  3. Watch the card transition through ENRICHED → INVESTIGATING → RESOLVED.
  4. Expand the tool-call trace to see which searches the agent ran and whether any write attempts were blocked by the guardrail.
- Card **How it works**: one paragraph on submit → enrich → investigate → resolve; one paragraph on the governance mechanism (before-tool-call guardrail on the write_ticket tool).
- Card **Components**: rows per component (BugEntity, TicketSyncConsumer, BugWorkflow, BugResolutionAgent, TicketWriteGuardrail, BugView, BugEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `decisions.authority_level = automated` declaration is filled and prominent. `oversight.human_in_loop = false` and `oversight.human_on_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a bug. <span class="accent">Read the resolution.</span>`. Subtitle: `One agent, one tool-call guardrail around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded example` (Java NPE / Go deadlock / Python async race / custom), `Title` text input, `Description` textarea, `Priority` select (LOW / MEDIUM / HIGH / CRITICAL), `Component` text input, `Reported by` text input, and a yellow `Submit bug` button.
    - Live list below: one card per bug, newest-first. Each card shows status pill, resolution badge (when resolved), confidence chip, bug title, age.
  - **Right column** — Selected-bug detail.
    - Header: status pill + resolution badge + confidence chip + bug title + ticket key.
    - Bug description: the submitted description text.
    - Ticket metadata: assignee, labels, project, filed timestamp.
    - Tool-call trace: a collapsible list of tool calls in invocation order. Each entry shows tool name, arguments summary, and outcome badge (`PASSED` in green / `REJECTED` in red). Rejected `write_ticket` entries show the guardrail's error message.
    - Resolution section (visible when `RESOLVED`): resolution body paragraph, confidence chip, evidence list (query + snippet + URL per search result used).
- Status pill colours: SUBMITTED=muted, ENRICHED=blue, INVESTIGATING=yellow, RESOLVED=green, FAILED=red.
- Resolution badge colours: FIXED=green, NEEDS_MORE_INFO=yellow, WONT_FIX=muted, DUPLICATE=blue.
- Confidence chip colours: HIGH=green, MEDIUM=yellow, LOW=red.
- Priority chip colours: LOW=muted, MEDIUM=blue, HIGH=yellow, CRITICAL=red.
