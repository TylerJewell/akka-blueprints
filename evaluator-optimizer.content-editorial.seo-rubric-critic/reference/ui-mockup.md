# UI mockup — seo-rubric-critic

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: SEO Article Evaluation</title>`.

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
- Headline: `SEO Article <span class="accent">Evaluation</span>`. **No subtitle.**
- Card **Try it** — one block: `/akka:build`. Then three numbered steps:
  1. Submit an article in the App UI tab (title, body, target keyword).
  2. Watch the article enter `SCORING`, then `REVISING`, then either `APPROVED` or `FAILED_FINAL`.
  3. Expand the article to see every round's draft, the guardrail verdict, the rubric score, and the failing dimension breakdown.
- Card **How it works** — one paragraph naming the evaluator-optimizer pattern: a rubric agent scores against a 21-point checklist, an optimizer agent revises specific sections, the workflow loops until the score clears the threshold or the round ceiling is reached.
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
- ID badges coloured per mechanism (guardrail = red, eval-event = blue, halt = red).
- Rows expand vertically on click; one open at a time. Expanded body shows the control's `rationale` paragraph and `implementation` paragraph from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an article. <span class="accent">Watch it get scored.</span>` Subtitle: `The simulator also drips an article brief every 60 s so the page is never empty.`
- Form card: text field labelled "Title", large textarea labelled "Article body", text field labelled "Target keyword", numeric field labelled "Word-count ceiling" (default 1500), `Submit` button (yellow).
- Live list: cards per article; left border coloured by status:
  - `SCORING` — blue
  - `REVISING` — orange
  - `APPROVED` — green
  - `FAILED_FINAL` — red
- Header row: title (first 60 chars), status pill, round counter (`n/maxRounds`), best-score chip showing highest composite score achieved.
- Click to expand: per-round timeline. Each round block shows:
  - **Round n** header with the timestamp.
  - The draft word count and a monospaced preview (first 300 characters).
  - The guardrail verdict pill (`OK` green, `OVER_WORD_CEILING` red with the detail text).
  - The rubric verdict pill (`PASS` green, `FAIL` orange) plus the composite score badge (e.g., `82/100`).
  - The failing dimensions list when `passed = false`: each row shows dimension name, score chip, and note.
  - The `improvementSummary` text when `passed = false`.
- Terminal block:
  - On `APPROVED`: the approved body text in a highlighted box plus a "passed at round N" caption and final composite score.
  - On `FAILED_FINAL`: the highest-scoring draft's body text plus the structured `rejectionReason` and round count.
