# UI mockup — meeting-assistant-zoom

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Zoom AI Meeting Assistant</title>`.

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
- Headline: `Zoom AI <span class="accent">Meeting Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded meetings (or paste your own transcript) and click **Process meeting**.
  3. Watch the card transition through TRANSCRIBING → TRANSCRIBED → SUMMARIZING → SUMMARIZED → DISPATCHING → DISPATCHED → EVALUATED.
  4. Inspect the redaction-audit badge on the transcript panel and the rejection-log strip if any scope rejections fired.
- Card **How it works**: one paragraph on the three task phases (TRANSCRIBE → SUMMARIZE → DISPATCH) and the typed handoff between them; one paragraph on the two governance mechanisms (PII sanitizer runs before the first LLM call; scope guardrail blocks misordered and out-of-scope writes).
- Card **Components**: rows per component (MeetingEntity, MeetingPipelineWorkflow, MeetingAgent, TranscribeTools, SummarizeTools, DispatchTools, ScopeGuardrail, PiiSanitizer, ActionItemScorer, MeetingView, MeetingEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, 1 Sanitizer, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is distinctive — transcripts contain real participant names and contact details. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the anchoring answers. `transcript-recording-consent-obtained` and `participant-pseudonymization-reversibility-policy` appear as `TO_BE_COMPLETED_BY_DEPLOYER` and are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 teal (sanitizer), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Upload a transcript. <span class="accent">Get a meeting package.</span>`. Subtitle: `One agent, three task phases, PII stripped before the first model call.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a `Meeting title` text input, a `Transcript` textarea (with a "Pick a seeded meeting" dropdown that fills both fields), and a yellow `Process meeting` button.
    - Live list below: one card per meeting, newest-first. Each card shows status pill, eval score chip (when eval landed), meeting title, age, a redaction count badge (e.g. "4 redactions"), and a small orange dot if any scope rejection fired.
  - **Right column** — Selected-meeting detail.
    - Header: status pill + eval score chip + meeting title.
    - Redaction-audit panel: shown immediately after `TranscribeStarted` lands (before the transcript result). Shows total redaction count and a collapsed table of `{ original → pseudonym, patternType }` entries. Expandable.
    - Phase panel 1 (Transcript segments): a table with columns speaker, text, startTime. Visible once `transcript.isPresent()`.
    - Phase panel 2 (Summary): summary paragraph, action-items list (actionId, description, assignee, dueDate), decisions list. Visible once `summary.isPresent()`.
    - Phase panel 3 (Meeting package): title, follow-up events list (title, attendees, date), tasks list (assignee, description, dueDate). Visible once `meetingPackage.isPresent()`.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the meeting has any `scopeRejections`): a small table with phase, tool, reason, time.
- Status pill colours: CREATED=muted, TRANSCRIBING=blue, TRANSCRIBED=blue, SUMMARIZING=yellow, SUMMARIZED=yellow, DISPATCHING=blue, DISPATCHED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A meeting in `SUMMARIZING` shows panels 1 and 2 (panel 2 with an "in progress" indicator if the agent has not yet returned). The redaction-audit panel appears as soon as `TranscribeStarted` lands — before the LLM returns — providing immediate evidence that PII was handled before the model call.
