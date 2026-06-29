# UI mockup — conversational-interview

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: ConversationalInterview</title>`.

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
- Headline: `Conversational<span class="accent">Interview</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded role (Software Engineer / Product Manager / Customer Success) or paste a custom role definition.
  3. Enter a candidate handle and click **Start session**.
  4. Enter each candidate answer when prompted; watch the session progress turn by turn to `COMPLETED`.
- Card **How it works**: one paragraph on start → first question → answer → screen → next question → complete; one paragraph on the two governance mechanisms (special-category sanitizer, bias/legality guardrail).
- Card **Components**: rows per component (InterviewSessionEntity, AnswerSanitizer, InterviewWorkflow, InterviewConductorAgent, QuestionGuardrail, SessionView, SessionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `special_category_personal_data: true` declaration in Data is filled and prominent. `decisions.authority_level = assist-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Conduct an interview. <span class="accent">One question at a time.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Session panel + live list.
    - Session panel: dropdown `Role` (Software Engineer / Product Manager / Customer Success / custom), `Candidate handle` text input (anonymised), `Started by` text input, and a yellow `Start session` button.
    - Live list below: one card per session, newest-first. Each card shows status pill, role badge, turn count, and age.
  - **Right column** — Selected-session detail.
    - Header: status pill + role badge + `candidateHandle`.
    - Competencies list: the role's competencies with ID, name, and description.
    - Turn-by-turn transcript: for each `TurnRecord` — the agent's question (labelled with the `competencyId` chip), the screening result (category chips + redacted preview if categories were found), and the candidate's answer (if submitted). The current open turn's question shows an **Answer** textarea and a **Submit answer** button.
    - Completion summary at bottom (visible once `COMPLETED`): turn count, competencies covered, session duration.
- Status pill colours: OPEN=blue, ANSWER_RECEIVED=muted, ANSWER_SCREENED=yellow, CONDUCTING=yellow, COMPLETED=green, FAILED=red.
- Competency chip: coloured accent, truncated to 16 chars.
- Protected-category chip colours: one distinct colour per category type (health=orange, religion=purple, pregnancy=pink, political=red, race=teal, union=blue, sexual-orientation=violet, financial=brown).

The raw candidate answer is never displayed on this screen — only the screened form. Coordinators who need the raw text fetch `/api/sessions/{id}` and read `turns[n].submittedAnswer.rawAnswer` from the JSON. This is intentional: the UI demonstrates that the model's input is the screened form, even though the audit trail keeps the raw.
