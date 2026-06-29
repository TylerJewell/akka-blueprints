# UI mockup — warehouse-optimizer

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from the blueprint-authoring guide.

Browser title: `<title>Akka Sample: Data Warehouse Optimizer</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Data Warehouse <span class="accent">Optimizer</span>`. **No subtitle.**
- Card **Try it** — one block: `/akka:build`. Then three numbered steps:
  1. Submit a slow query or DDL statement in the App UI tab.
  2. Watch the request progress through `PROPOSING` → `EVALUATING` → `APPROVED` (or pause at `AWAITING_DBA` for DDL changes).
  3. Expand the request to see every attempt's proposal, the guardrail verdict, the DBA decision (if applicable), and the evaluator's verdict and notes.
- Card **How it works** — one paragraph naming the evaluator-optimizer pattern: an optimizer proposes query rewrites or schema changes, an evaluator scores them, the workflow loops until convergence or the attempt ceiling; DDL proposals are intercepted before the evaluator and require DBA approval.
- Card **Components** — table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract** — table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (2 agents, 1 workflow, 2 entities, 1 view, 1 consumer, 3 timed actions, 2 endpoints).
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
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` block rendered with the question text, drives sublabel, and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks (where the YAML value matches `/TO_BE_COMPLETED_BY_DEPLOYER/i`) get `opacity: 0.45` and a muted-italic placeholder ("To be completed by deployer").

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: guardrail = red (`G1`), hitl = orange (`H1`), eval-event = blue (`E1`), halt = red (`HT1`).
- Rows expand vertically on click; one open at a time. Expanded body shows the control's `rationale` paragraph and `implementation` paragraph from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a query. <span class="accent">Watch it get optimized.</span>` Subtitle: `The simulator also drips a request every 60 s so the page is never empty.`
- Form card: textarea labelled "SQL Statement" (monospace, 4 rows), text field labelled "Objective" (default "improve performance"), `Submit` button (yellow).
- Live list: cards per request; left border coloured by status:
  - `PROPOSING` — muted grey
  - `AWAITING_DBA` — amber/orange
  - `EVALUATING` — blue
  - `APPROVED` — green
  - `REJECTED_FINAL` — red
- Header row: first 80 chars of `originalSql`, status pill, attempt counter (`n/maxAttempts`), best-score chip.
- Click to expand: per-attempt timeline. Each attempt block shows:
  - **Attempt n** header with the timestamp.
  - The proposed SQL in a monospaced block with syntax highlighting for keywords.
  - The proposal kind badge (`QUERY_REWRITE`, `INDEX_ADD`, etc.).
  - The guardrail verdict pill (`OK` green, `DDL_DETECTED` amber).
  - When status is `AWAITING_DBA`: an inline DBA decision form with Approve / Reject buttons, a text field for decision note, and a submitter name field.
  - The DBA decision verdict pill (`APPROVED` green, `REJECTED` red) plus the decision note and decidedBy, when a decision has been recorded.
  - The evaluator's verdict pill (`APPROVE` green, `REVISE` orange) plus the score badge.
  - The evaluator's notes (overallRationale + bullets) when present.
- Terminal block:
  - On `APPROVED`: the approved SQL in a highlighted monospace box plus a "best of N attempts" caption.
  - On `REJECTED_FINAL`: the best-scoring proposal's SQL plus the structured `rejectionReason`.
