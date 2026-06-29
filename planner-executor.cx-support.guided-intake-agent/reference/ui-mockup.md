# UI mockup — guided-intake-agent

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from Section 13 of the blueprint authoring guide.

Browser title: `<title>Akka Sample: Guided Intake Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index.

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient (Lesson 26).

## Mermaid CSS overrides (Lesson 24)

The `<style>` block in `static-resources/index.html` includes the CSS overrides AND `themeVariables` from Lesson 24 — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state-machine diagram on the Architecture tab renders state names invisible and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Guided <span class="accent">Intake Agent</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — select a goal and start a session in the App UI tab, answer the agent's questions, expand the row to see the plan, the turn log, and the filled summary.
- Card **How it works**: one paragraph naming the components, the loop steps (plan → propose → guardrail → ask → await-reply → sanitize → record → evaluate), and the two agents.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/conversations/{id}/reply` route and the `/api/control/*` operator routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (2 agents, 1 workflow, 4 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state machine, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text, drives sublabel, and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `sanitizer` green (`S1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Start an intake session. <span class="accent">Answer turn by turn.</span>` Subtitle: `The simulator drips a session every 90 s so the page is never empty.`
- **Session form card**: goal selector (populated from `GET /api/goals`), optional submitter field, `Start session` button (yellow).
- **Operator controls pane** (top right of the App UI tab):
  - If `halted=false`: yellow `Halt new sessions` button, a free-text reason field.
  - If `halted=true`: muted `HALTED` pill with reason and timestamp, plus a `Resume` button.
  - Pane reflects every `control-update` SSE event live.
- **Active conversation pane** (centre): shows the current open question delivered via `question-ready` SSE, a reply text area, and a `Send reply` button. Appears only when there is an `ELICITING` conversation waiting for a reply from this browser session.
- **Live list**: cards per conversation; left border coloured by status (PLANNING = muted, ELICITING = blue, COMPLETED = green, ESCALATED = orange, HALTED = orange, ABANDONED = pale red).
  - Header row: goal name, status pill, turn count, elapsed time.
  - Click to expand:
    - **Plan**: goal name, ordered question list with required markers.
    - **Turn log**: vertical timeline of `TurnRecord` entries. Each row shows turn number, question text (truncated to 80 chars), verdict pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `INCOMPLETE` muted, `ESCALATED` orange), and a collapsed `<pre>` of `sanitizedReply` (click to expand). Redacted spans render in italics with a tooltip showing the redaction tag (e.g., `[REDACTED:email]`).
    - **Summary** (only when status is `COMPLETED`): filled-fields table (field name → sanitized answer) + confidence bar.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
