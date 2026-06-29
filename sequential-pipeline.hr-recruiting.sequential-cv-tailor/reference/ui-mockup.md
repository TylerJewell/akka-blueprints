# UI mockup — sequential-cv-tailor

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Sequential CV Tailor</title>`.

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
- Headline: `Sequential CV <span class="accent">Tailor</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a candidate and a job posting from the dropdown lists, then click **Tailor CV**.
  3. Watch the card transition through GENERATING → GENERATED → TAILORING → TAILORED → SCORED.
  4. Review the keyword-match chips and alignment score on the right pane. A red border means a required keyword was not found.
- Card **How it works**: one paragraph on the two-stage pipeline (generate base CV from candidate profile, then tailor it for a specific job posting) and the typed handoff between stages; one paragraph on the two governance mechanisms (PII sanitizer on every model call, alignment eval after tailoring).
- Card **Components**: rows per component (CvRequestEntity, CvPipelineWorkflow, CvGeneratorAgent, CvTailorAgent, ProfileTools, TailorTools, PiiSanitizer, AlignmentScorer, CvRequestView, CvEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 AutonomousAgents, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 2 function-tool classes, 1 Sanitizer, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and highlighted (candidate profiles contain names, emails, and dates of birth). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, E1). ID badges coloured: S1 orange (sanitizer), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Pick a candidate and posting. <span class="accent">Get a tailored CV.</span>`. Subtitle: `Two agents, one PII gate before each model call, one alignment check after.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: two dropdowns (`Candidate` filled from seeded fixture list, `Job Posting` filled from seeded fixture list) and a yellow `Tailor CV` button.
    - Live list below: one card per request, newest-first. Each card shows status pill, alignment score chip (when scored), candidate name alias (first name only — full name not displayed), posting title, age, and a small orange lock icon if any sanitization events fired.
  - **Right column** — Selected-request detail.
    - Header: status pill + alignment score chip + candidate alias + posting title.
    - Stage panel 1 (Base CV): generated summary paragraph, skills table with level badges, experience entries. Visible once `baseCv` is present.
    - Stage panel 2 (Tailored CV): tailored summary paragraph, keyword-match chips (green = found, red = required but missing, grey = preferred but missing), adjusted experience entries. Visible once `tailoredCv` is present.
    - Alignment section at bottom: a 1–5 bar widget, the one-line rationale, and two chip rows (Keywords covered / Keywords missed).
    - Sanitization-audit strip: always visible once the request exists; shows the count of sanitization events and a tooltip listing `agent` + `removedCount` per event.
- Status pill colours: CREATED=muted, GENERATING=blue, GENERATED=blue, TAILORING=yellow, TAILORED=yellow, SCORED=green, FAILED=red.

Each stage panel renders only when its data is present on the row record. A request in `TAILORING` shows Stage 1 (BaseCv) and Stage 2 with an "in progress" spinner until the tailor agent returns. This is the visual proof that the typed handoff is the only path candidate data travels.
