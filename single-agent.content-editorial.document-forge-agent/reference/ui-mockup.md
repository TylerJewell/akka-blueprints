# UI mockup — document-forge-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Document Forge</title>`.

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

And the DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Document <span class="accent">Forge</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a document type and style template, then type your prompt — or click **Load seeded example** to fill all three.
  3. Click **Forge document**.
  4. Watch the card transition through FORGING → FORGE_COMPLETED → AUDITED.
- Card **How it works**: one paragraph on submit → forge → audit; one paragraph on the guardrail mechanism (before-tool-call path validation and content guardrail).
- Card **Components**: rows per component (ForgeEntity, ForgeOutputConsumer, ForgeWorkflow, DocumentForgeAgent, WriteDocumentGuardrail, ForgeAuditor, ForgeView, ForgeEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Auditor as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `path_validated_before_write: true` declaration in Data is filled and prominent. `decisions.authority_level = autonomous` and `oversight.human_in_loop = false` are the distinctive answers — many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Describe it. <span class="accent">Forge it.</span>`. Subtitle: `One agent, one guardrail between intent and the file store.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Document type` (Business Memo / Technical Brief / Marketing Copy / Project Proposal / Custom), dropdown `Style template` (Formal / Casual / Technical / Executive Summary), `Prompt` textarea (with a "Load seeded example" link that fills all fields), `Requested by` text input, and a yellow `Forge document` button.
    - Live list below: one card per forge, newest-first. Each card shows status pill, audit score chip (when audit landed), document type label, age.
  - **Right column** — Selected-forge detail.
    - Header: status pill + audit score chip + document type label.
    - Prompt section: the original prompt text, style template chip.
    - Generated document panel: a monospace-ish block of the document content, with the output filename above it and a copy-to-clipboard button.
    - Audit section at bottom: a 1–5 star widget, the one-line assessment, and a list of any flagged patterns. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, FORGING=yellow, FORGE_COMPLETED=blue, AUDITED=green, FAILED=red.
- Audit score chip: 1–2=red, 3=yellow, 4–5=green.
- Document type chip: colour-coded by type (memo=blue, brief=purple, copy=orange, proposal=teal, custom=muted).
