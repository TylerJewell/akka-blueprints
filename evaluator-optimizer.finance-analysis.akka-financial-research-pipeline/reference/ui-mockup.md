# UI mockup — financial-research-pipeline

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from the blueprint authoring guide.

Browser title: `<title>Akka Sample: Financial Research Multi-Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Financial Research <span class="accent">Multi-Agent</span>`. **No subtitle.**
- Card **Try it** — one block: `/akka:build`. Then numbered steps:
  1. Submit a research query in the App UI tab.
  2. Watch the report progress through `PLANNING` → `RESEARCHING` → `ANALYSING` → `WRITING` → `VERIFYING`.
  3. Approve the report in the compliance queue to move it to `APPROVED`.
  4. Expand the report card to see every sub-task, source bundle summary, analysis sections, drafts, and verification verdicts.
- Card **How it works** — one paragraph describing the evaluator-optimizer pattern: a planner decomposes the query, a search agent retrieves sources, an analyst synthesises findings, a writer drafts the report, and a verifier quality-checks until convergence or ceiling exhaustion.
- Card **Components** — table with rows for each component from `SPEC.md §4`. Kind column coloured per the component palette.
- Card **API contract** — table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: 5 agents, 1 workflow, 2 entities, 1 view, 1 consumer, 2 timed actions, 2 endpoints.
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables AND the Lesson 24 CSS overrides:

  ```css
  .diagram-card .mermaid .nodeLabel,
  .diagram-card .mermaid .stateLabel,
  .diagram-card .mermaid g.statediagram-state .label,
  .diagram-card .mermaid g.statediagram-state .label *,
  .diagram-card .mermaid g.statediagram-state text,
  .diagram-card .mermaid g.node text,
  .diagram-card .mermaid .label foreignObject div,
  .diagram-card .mermaid .label foreignObject p {
    color:#ffffff !important; fill:#ffffff !important;
  }
  .diagram-card .mermaid .edgeLabel foreignObject,
  .diagram-card .mermaid g.edgeLabel foreignObject,
  .diagram-card .mermaid g.edgeLabels foreignObject { overflow:visible !important; }
  .diagram-card .mermaid .edgeLabel foreignObject > div {
    white-space:nowrap !important; overflow:visible !important; display:inline-block !important;
  }
  ```

  Plus `mermaid.initialize({themeVariables: { nodeTextColor: '#fff', stateLabelColor: '#fff', transitionLabelColor: '#cccccc', ... }})`.

- Compressed comp-row table: one row per component, click expands to show description plus a short Java source snippet (10–20 lines) with `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` block rendered with the question text and answer chips/textareas per `risk-survey.yaml`.
- Unanswered `.qb` blocks (YAML value matches `/TO_BE_COMPLETED_BY_DEPLOYER/i`) get `opacity: 0.45` and a muted-italic placeholder.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badge colours: eval-event = blue, sanitizer = orange, guardrail = red, hotl = purple.
- Rows expand vertically on click; one open at a time. Expanded body shows the control's `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a query. <span class="accent">Watch a report land.</span>` Subtitle: `The simulator drips a sample query every 90 s so the page is never empty.`
- Form card: text field labelled "Research query", select labelled "Sector" (equity / credit / macro / commodities / general), numeric field labelled "Word ceiling" (default 800), `Submit` button (yellow).
- Live list: cards per report; left border coloured by status:
  - `PLANNING` — muted grey
  - `RESEARCHING` — blue
  - `ANALYSING` — teal
  - `WRITING` — orange
  - `VERIFYING` — blue
  - `AWAITING_COMPLIANCE` — purple
  - `APPROVED` — green
  - `REJECTED_FINAL` — red
- Header row: topic (first 70 chars), status pill, draft counter (`n/maxVerifyAttempts`), best-score chip.
- Click to expand: per-draft timeline. Each draft block shows:
  - **Draft n** header with timestamp.
  - The guardrail verdict pill (`OK` green, `OVER_WORD_CEILING` red with detail text).
  - The verifier's verdict pill (`APPROVE` green, `REVISE` orange) plus score badge.
  - The verifier's notes (`overallRationale` + bullets) when present.
- Sanitiser log panel: visible when `sanitizerLog.changed = true`; lists removed items.
- Compliance queue panel: visible when `status = AWAITING_COMPLIANCE`. Shows an "Approve" and "Reject" button that call the compliance endpoints; disabled when status is terminal.
- Terminal block:
  - On `APPROVED`: the approved text in a highlighted box plus "best of N drafts."
  - On `REJECTED_FINAL`: the best-scoring draft's text plus the `rejectionReason`.
