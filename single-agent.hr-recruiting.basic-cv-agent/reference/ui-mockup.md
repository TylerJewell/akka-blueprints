# UI mockup — basic-cv-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: BasicCvAgent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The exemplar's `static-resources/index.html` has the canonical implementation:

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
- Headline: `Basic<span class="accent">CvAgent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded candidate profile (Software Engineer / Marketing Manager / Recent Graduate) or fill in the form manually.
  3. Select an output mode (Prose CV or Structured CV) and click **Generate CV**.
  4. Watch the card transition through SANITIZED → GENERATING → CV_GENERATED and read the output.
- Card **How it works**: one paragraph on submit → sanitize → generate; one paragraph on the single governance mechanism (PII sanitizer removes identifiers before the LLM call; raw profile is preserved on the entity for audit).
- Card **Components**: rows per component (CvEntity, ProfileSanitizer, CvWorkflow, CvGeneratorAgent, CvView, CvEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (S1). ID badge coloured green (sanitizer).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a profile. <span class="accent">Get a CV.</span>`. Subtitle: `One agent, one sanitizer around it.`
- Layout: two-column.
  - **Left column** — Submission form + live list.
    - Submission form at top: **Candidate** section with Full name text input, Contact email text input, Target role text input; **Experience** section with an add-entry button for work history entries (each entry: title, company, start date, end date or "present" toggle, bullet list); **Education** section with add-entry button (each entry: degree, institution, graduation year); **Skills** text input (comma-separated); **Additional notes** textarea (placeholder: "Certifications, languages, availability..."); **Output mode** toggle (`Prose CV` / `Structured CV`); **Submitted by** text input; a yellow **Generate CV** button. Below the form: a **Load seeded profile** dropdown (Software Engineer / Marketing Manager / Recent Graduate) that pre-fills the entire form.
    - Live list below form: one card per request, newest-first. Each card shows status pill, output mode badge, candidate name, age.
  - **Right column** — Selected-request detail.
    - Header: status pill + output mode badge + candidate name + target role.
    - Sanitized profile preview: collapsed section showing the sanitized notes and the redacted contact email, with PII category chips above (`email`, `nino`, `ssn`, etc.).
    - Generated CV output: for Prose mode, a rendered Markdown block with section headings; for Structured mode, a formatted JSON tree with syntax highlighting and a copy-to-clipboard button.
    - Keywords list: a row of chip tags below the CV output.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, GENERATING=yellow, CV_GENERATED=green, FAILED=red.
- Output mode badge colours: PROSE=teal, STRUCTURED=purple.

The raw contact email and additional notes are never displayed on this screen — only the sanitized forms. Recruiters who need the raw data fetch `GET /api/cv-requests/{id}` and read `request.contactEmail` and `request.additionalNotes` from the JSON.
