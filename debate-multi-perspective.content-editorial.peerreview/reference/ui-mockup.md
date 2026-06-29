# UI mockup — peerreview

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Peer Review Panel</title>`.

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

And the DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents (clicking a tab and seeing a blank because a hidden zombie panel occupied the index).

## Mermaid theming — MUST carry the Lesson 24 overrides

The Architecture tab's `stateDiagram-v2` needs explicit CSS so state names render white-on-dark and edge labels are not clipped. The `<style>` block must set `color`/`fill` to `#ffffff` on every state-label DOM path and `overflow:visible` on every edge-label `foreignObject`, and `mermaid.initialize` must set `nodeTextColor`, `stateLabelColor`, and `transitionLabelColor: #cccccc`. Without both, state names render black-on-black and transition labels clip at the top.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Peer <span class="accent">Review Panel</span>`. **No subtitle.**
- Card **Try it**: just the `/akka:build` (Claude Code) block — no env-var export block — then three numbered steps (submit a document, watch the review progress to SYNTHESISED, expand it for the three axis reviews + overall verdict + redaction count).
- Card **How it works**: one paragraph naming the components and the parallel three-reviewer fan-out reconciled by the moderator.
- Card **Components**: table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (5 agents, 1 workflow, 2 entities, 1 view, 1 consumer, 2 timed actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables and the Lesson 24 overrides.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text, drives sublabel, and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer"); unanswered `.qb` blocks get `opacity: 0.45`.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: sanitizer = green, guardrail = red, eval-event = blue.
- Rows expand vertically on click; one open at a time, showing rationale + implementation from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a document. <span class="accent">Watch a panel review it.</span>` Subtitle: `The simulator also drips a document every 60 s so the page is never empty.`
- Form card: a text field labelled "Title", a textarea labelled "Document body", a `Submit` button (yellow). Helper text under the body field: "Personal data is redacted before any reviewer reads it."
- Live list: cards per review; left border coloured by status (INTAKE = muted, REVIEWING = blue, SYNTHESISED = green, DEGRADED = orange, BLOCKED = red).
  - Header row: title, status pill, a redaction-count chip (e.g. "3 redacted"), and an optional consistency-score chip.
  - Click to expand: three axis-review blocks (Technical / Style / Compliance) each showing the axis verdict, score, and findings list with severity badges; then the overall verdict block (verdict + summary + guardrail verdict); then the consistency rationale if scored. The redacted body is shown in a collapsed `<details>` so reviewers can confirm the redaction without the raw values.
