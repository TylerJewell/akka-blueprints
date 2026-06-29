# UI mockup — sales-roleplay-coach

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Sales Roleplay Coach</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Sales Roleplay <span class="accent">Coach</span>`. **No subtitle.**
- Card **Try it** — one block: `/akka:build`. Then three numbered steps:
  1. Submit a practice scenario in the App UI tab (buyer persona, product, deal stage).
  2. Watch the session open with the buyer's first move, then submit your pitch turns.
  3. Expand the session to see each turn's guardrail verdict, the buyer's reaction, and the coach's scoring notes.
- Card **How it works** — one paragraph naming the evaluator-optimizer pattern: a buyer simulator responds to each rep pitch, a coach scores the turn, the workflow loops until the coach accepts or the attempt ceiling is reached.
- Card **Components** — table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract** — table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (2 agents, 1 workflow, 2 entities, 1 view, 1 consumer, 2 timed actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state machine, ER) with Akka theme variables AND the Lesson 24 CSS overrides:
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
- ID badges coloured per mechanism (guardrail = red, eval-event = blue).
- Rows expand vertically on click; one open at a time. Expanded body shows the control's `rationale` paragraph and `implementation` paragraph from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a scenario. <span class="accent">Sharpen your pitch.</span>` Subtitle: `The simulator also drips a scenario every 90 s so the page is never empty.`
- Form card: text field labelled "Buyer persona", text field labelled "Product", dropdown labelled "Deal stage" (values: DISCOVERY, DEMO, PROPOSAL, NEGOTIATION, CLOSING; default DISCOVERY), optional numeric field labelled "Deal amount (USD)", `Start session` button (yellow).
- After a session is created, a **pitch input area** appears at the bottom of the active session card: a text area labelled "Your pitch turn" and a `Submit turn` button.
- Live list: cards per session; left border coloured by status:
  - `PITCHING` — muted grey
  - `EVALUATING` — blue
  - `PASSED` — green
  - `EXHAUSTED` — red
- Header row: buyer persona (first 50 chars), product, deal stage chip, status pill, turn counter (`n/maxTurns`), best-score chip.
- Click to expand: per-turn timeline. Each turn block shows:
  - **Turn n** header with the timestamp.
  - The rep's pitch text in a monospaced block.
  - The guardrail verdict pill (`OK` green, `PROHIBITED_CONTENT` red with the detail text).
  - The buyer's response text (when present) with the signal badge (`INTERESTED` green, `SKEPTICAL` yellow, `RESISTANT` orange, `READY_TO_BUY` green).
  - The coach's decision pill (`ACCEPT` green, `REVISE` orange) plus the score badge.
  - The coaching notes (overallRationale + bullets) when present.
- Terminal block:
  - On `PASSED`: the accepted pitch text in a highlighted box plus a "passed on turn N of maxTurns" caption.
  - On `EXHAUSTED`: the highest-scoring turn's text plus the structured `exhaustionReason`.
