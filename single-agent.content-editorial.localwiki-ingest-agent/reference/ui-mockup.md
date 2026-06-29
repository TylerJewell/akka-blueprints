# UI mockup — localwiki-ingest-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: LocalWiki Demo Agent</title>`.

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
- Headline: `LocalWiki <span class="accent">Demo Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Select a source type (URL / Image / PDF) and a target namespace, or click a seeded example.
  3. Click **Submit for ingest**.
  4. Watch the card transition through SANITIZED → INGESTING → PAGE_FILED.
- Card **How it works**: one paragraph on submit → fetch/sanitize → agent call → page filed; one paragraph on the two governance mechanisms (path-safety guardrail, PII sanitizer).
- Card **Components**: rows per component (IngestEntity, ContentSanitizer, IngestWorkflow, WikiIngestAgent, WritePageGuardrail, WikiPageView, IngestEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = autonomous` and `oversight.human_on_loop = true` are the distinctive answers (contrasting with a recommend-only system). Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a source. <span class="accent">File a wiki page.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: `Source type` radio group (URL / Image / PDF), `Source address` text input, `Target namespace` dropdown (`/docs` / `/blog` / `/reference` / custom), `Submitted by` text input, and a "Load seeded example" link (populates both source address and namespace), and a yellow `Submit for ingest` button.
    - Live list below: one card per ingest, newest-first. Each card shows status pill, source type badge, age, and the ingest ID.
  - **Right column** — Selected-ingest detail.
    - Header: status pill + source type badge + `ingestId`.
    - Source address: the submitted address (read-only).
    - Sanitized content preview: a monospace block of the redacted source text, with PII category chips above (`email`, `phone`, `ssn`, etc.).
    - Filed page section (visible once `PAGE_FILED`): page title as a heading, target path chip, one-paragraph summary, body rendered as markdown, category tag chips.
    - Failure section (visible on `FAILED`): failure reason string in a red callout.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, INGESTING=yellow, PAGE_FILED=green, FAILED=red.
- Source type badge colours: URL=muted-teal, IMAGE=muted-purple, PDF=muted-orange.
- PII category chip colours: all muted-red.

The raw fetched text is never displayed on this screen — only the sanitized form. Editors who need the raw source fetch `/api/ingests/{id}` and read `submission.sourceAddress` from the JSON.
