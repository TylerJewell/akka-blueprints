# UI mockup — mr-reviewer

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: GitLab MR Reviewer</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `GitLab MR <span class="accent">Reviewer</span>`. **No subtitle.**
- Card **Try it** — one block: `/akka:build`. Then four numbered steps:
  1. Submit a diff webhook in the App UI tab.
  2. Watch the MR pass through the sanitizer, enter `REVIEWING`, then `GATE_CHECKING`, then either `CI_PASS` or `CI_FAIL`.
  3. Expand the MR to see each pass's findings, guardrail verdict, gate score, and gate feedback.
  4. Check `GET /api/mrs/{id}/ci-signal` for the machine-readable terminal signal.
- Card **How it works** — one paragraph naming the evaluator-optimizer pattern: a reviewer agent reads the diff and produces findings; a gate agent scores thoroughness and actionability; the workflow loops until the gate passes or the retry ceiling is reached. Two deterministic gatekeeping steps — a secret sanitizer before any LLM call, and a commentary guardrail before the gate — protect against credential leakage.
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
- ID badges coloured per mechanism: sanitizer = red, guardrail = red, ci-gate = blue.
- Rows expand vertically on click; one open at a time. Expanded body shows the control's `rationale` paragraph and `implementation` paragraph from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a diff. <span class="accent">Watch a review land.</span>` Subtitle: `The simulator also drips a webhook every 90 s so the page is never empty.`
- Form card: text field labelled "Project path (e.g. acme/backend)", numeric field labelled "MR IID", text field labelled "Target branch" (default `main`), textarea labelled "Diff text", `Submit` button (yellow).
- Live list: cards per MR; left border coloured by status:
  - `RECEIVED` — muted grey
  - `REVIEWING` — blue
  - `GATE_CHECKING` — purple
  - `CI_PASS` — green
  - `CI_FAIL` — red
  - `SANITIZER_BLOCKED` — orange
- Header row: project path + MR IID, status pill, pass counter (`n/maxPasses`), CI signal chip.
- Click to expand: per-pass timeline. Each pass block shows:
  - **Pass n** header with the timestamp.
  - The sanitizer verdict pill (`OK` green, `SECRET_DETECTED` orange with the detail text).
  - The guardrail verdict pill (`OK` green, `COMMENTARY_REPRODUCES_SECRET` red with detail) if triggered.
  - Per-file findings table: file path, line, severity badge, message.
  - The review summary in a readable block.
  - Quality score badge.
  - Gate decision pill (`PASS` green, `REFINE` orange) plus gate score badge.
  - Gate feedback bullets when present.
- Terminal block:
  - On `CI_PASS`: the accepted review's summary in a highlighted box plus "accepted on pass N" caption; CI signal chip green.
  - On `CI_FAIL`: the best-scoring pass's summary plus the structured failure reason; CI signal chip red.
  - On `SANITIZER_BLOCKED`: a red banner showing the `SECRET_DETECTED` detail and noting that no LLM call was made.
