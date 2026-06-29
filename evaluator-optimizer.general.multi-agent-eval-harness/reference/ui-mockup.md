# UI mockup — multi-agent-eval-harness

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Multi-Agent Evaluation Harness</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Multi-Agent <span class="accent">Evaluation Harness</span>`. **No subtitle.**
- Card **Try it** — one block: `/akka:build`. Then three numbered steps:
  1. Submit a scenario suite in the App UI tab.
  2. Watch the run enter `RUNNING`, then `JUDGING`, then either `PASSED` or `FAILED`.
  3. Expand the run to see every scenario result, the threshold verdict, and the judge's notes.
- Card **How it works** — one paragraph naming the evaluator-optimizer pattern: an orchestrator fans scenarios to specialist agents, a judge scores the aggregate, the workflow loops until a passing verdict or the retry ceiling; the CI gate converts the verdict into a deployment signal.
- Card **Components** — table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract** — table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (2 agents, 1 workflow, 2 entities, 1 view, 1 consumer, 2 timed actions, 2 endpoints).
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
- ID badges coloured per mechanism (eval-periodic = blue, ci-gate = red).
- Rows expand vertically on click; one open at a time. Expanded body shows the control's `rationale` paragraph and `implementation` paragraph from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a suite. <span class="accent">Watch a verdict land.</span>` Subtitle: `The loader also submits a run every 120 s so the page is never empty.`
- Form card: text field labelled "Suite name", numeric field labelled "Pass threshold" (default 0.75, range 0.0–1.0 step 0.01), `Submit` button (yellow).
- Live list: cards per run; left border coloured by status:
  - `RUNNING` — muted grey
  - `JUDGING` — blue
  - `PASSED` — green
  - `FAILED` — red
- Header row: suite name, status pill, retry counter (`n/maxRetries`), final score chip (or "—" while in progress), CI gate badge (`BLOCKED` red when `ciGateBlocked = true`).
- Click to expand: per-scenario timeline. Each scenario block shows:
  - **Scenario scenarioId** header with `category` tag and `completedAt` timestamp.
  - The agent response in a monospaced block.
  - A threshold verdict pill (`MET` green, `NOT MET` red).
- Judgment block (rendered after the scenario list):
  - The judge's verdict pill (`PASS` green, `FAIL` red) plus the score badge.
  - The judgment notes (`overallRationale` + bullets) when present.
- Terminal block:
  - On `PASSED`: the final score in a highlighted box plus "run accepted" caption.
  - On `FAILED`: the best-scoring judgment's rationale plus the structured `failureReason`.
  - CI gate alert bar when `ciGateBlocked = true`: red banner "CI gate blocked — score below threshold".
