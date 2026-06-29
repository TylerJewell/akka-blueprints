# UI mockup — pitch-builder

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Pitch Builder</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents (clicking a tab and seeing a blank because a hidden zombie panel occupied the index).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Pitch <span class="accent">Builder</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded targets (or type your own) and click **Build pitchbook**.
  3. Watch the card transition through RESEARCHING → RESEARCHED → COMPS_RUNNING → COMPS_READY → DRAFTING → DRAFTED → VALIDATED.
  4. Inspect the PII-redaction badge on the research panel and the citation-rejection log if any guardrail rejections fired.
- Card **How it works**: one paragraph on the three task phases (RESEARCH → COMPARABLES → DRAFT), the PII sanitizer that runs at the research boundary, and the typed handoff between phases; one paragraph on the two governance mechanisms (PII sanitizer, citation guardrail).
- Card **Components**: rows per component (PitchbookEntity, PitchbookPipelineWorkflow, PitchAgent, ResearchTools, ComparablesTools, DraftTools, PiiSanitizer, CitationValidator, PitchbookView, PitchbookEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 PII Sanitizer, and 1 Citation Validator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled (raw research items may contain counterparty contact names and emails). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, H1). ID badges coloured: S1 green (sanitizer), H1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Pick a target. <span class="accent">Read the pitchbook.</span>`. Subtitle: `One agent, three task phases, PII sanitized at the boundary.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Target company` (with a "Pick a seeded target" dropdown that fills it), and a yellow `Build pitchbook` button.
    - Live list below: one card per pitchbook, newest-first. Each card shows status pill, validation score chip (when validation landed), target name, age, a small green lock icon if PII was redacted during research, and a small red dot if any citation rejection fired.
  - **Right column** — Selected-pitchbook detail.
    - Header: status pill + validation score chip + target name.
    - Phase panel 1 (Research items): a table with columns source, url, headline, and a redaction-count badge (e.g. `1 PII redacted`). Visible once `research.isPresent()`.
    - Phase panel 2 (Comparables): a peers table (ticker, name, sector, market cap) and a multiples grid (EV/EBITDA range, EV/Revenue range, P/E range). Visible once `comps.isPresent()`.
    - Phase panel 3 (Pitchbook): cover page metadata (target, sector, prepared by), executive summary paragraph, then a per-section block (heading, body, cited-tickers chips, cited-figures chips). Visible once `pitchbook.isPresent()`.
    - Validation section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Citation-rejection log strip (only visible if `citationRejections` is non-empty): a small table with sectionId, rule, detail, time.
- Status pill colours: CREATED=muted, RESEARCHING=blue, RESEARCHED=blue, COMPS_RUNNING=yellow, COMPS_READY=yellow, DRAFTING=blue, DRAFTED=blue, VALIDATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A pitchbook in `COMPS_RUNNING` shows panel 1 (research) and panel 2 with a "building comps" spinner. This is the visual proof that the typed handoff between phases is the only path information travels.
