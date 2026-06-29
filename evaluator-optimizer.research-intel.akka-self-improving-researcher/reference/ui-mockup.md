# UI mockup — self-improving-deep-researcher

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from the blueprint authoring guide.

Browser title: `<title>Akka Sample: Self-Improving Deep Research Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See the blueprint authoring guide Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Self-Improving Deep <span class="accent">Research Agent</span>`. **No subtitle.**
- Card **Try it** — one block: `/akka:build`. Then three numbered steps:
  1. Submit a research query in the App UI tab.
  2. Watch the session move through `RESEARCHING`, then `EVALUATING`, then either `ACCEPTED` or `MAX_ATTEMPTS_REACHED`.
  3. Expand the session to see every attempt's report summary, the evaluator's verdict and notes, and the memory diff the agent applied after the session ended.
- Card **How it works** — one paragraph naming the evaluator-optimizer pattern: a research agent synthesizes findings, an evaluator scores quality, the workflow loops until the report is sufficient or the attempt ceiling is reached, then a prompt-rewriter agent updates the system's own memory blocks to improve the next session.
- Card **Components** — table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract** — table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (3 agents, 1 workflow, 2 entities, 1 view, 1 consumer, 2 timed actions, 2 endpoints).
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
- ID badges coloured per mechanism (eval-periodic = blue, ci-gate = yellow, hotl = green).
- Rows expand vertically on click; one open at a time. Expanded body shows the control's `rationale` paragraph and `implementation` paragraph from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a query. <span class="accent">Watch an agent improve.</span>` Subtitle: `The simulator drips a query every 90 s so the page is never empty.`
- Form card: text field labelled "Research query", select labelled "Depth" (options: Brief / Standard / Deep, default Standard), `Submit` button (yellow).
- Live list: cards per session; left border coloured by status:
  - `RESEARCHING` — muted grey
  - `EVALUATING` — blue
  - `ACCEPTED` — green
  - `MAX_ATTEMPTS_REACHED` — red
- Header row: topic (first 70 chars), status pill, attempt counter (`n/maxAttempts`), best-score chip.
- Click to expand: per-attempt timeline. Each attempt block shows:
  - **Attempt n** header with the timestamp.
  - The executive summary in a prose block.
  - The evidence count chip (`n evidence segments`).
  - The evaluator's verdict pill (`SUFFICIENT` green, `REFINE` orange) plus the score badge.
  - The evaluator's notes (overallRationale + bullets) when present.
- Memory diff block (always shown when session is in a terminal state):
  - A "Memory updated" header with the diff timestamp.
  - Per-change rows: block ID chip, change type badge (`UPDATE` / `APPEND` / `NO_CHANGE`), the reason sentence.
  - Expandable before/after content for `UPDATE` and `APPEND` changes.
- Drift timeline (collapsible panel at the bottom of each expanded session):
  - Lists any `PromptDriftRecorded` events detected since the session was created.
  - Shows old fingerprint → new fingerprint, changed block count, and detection timestamp.
