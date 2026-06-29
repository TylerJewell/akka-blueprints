# UI mockup — http-content-agent

Five-tab structure inherited from the formal exemplar. Browser title:
`<title>Akka Sample: Claude AI Content Generator</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel`
`<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler
MUST switch tabs by matching these attributes — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture,
Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration
must be deleted from the HTML; `display:none` is not enough. (Lesson 26)

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Claude AI Content <span class="accent">Generator</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered
  steps:
  1. Open the App UI tab.
  2. Pick a seeded brief (technology trends / social campaign / email teaser) or fill in your
     own topic and target audience.
  3. Click **Generate**.
  4. Watch the card transition through GENERATING → APPROVED and read the finished content.
- Card **How it works**: one paragraph on submit → generate → guardrail check → approve; one
  paragraph on the governance mechanism (before-agent-response guardrail, brand-safety checks,
  retry on rejection).
- Card **Components**: rows per component (ContentJobEntity, ContentWorkflow,
  ContentGeneratorAgent, DraftGuardrail, ContentView, ContentEndpoint, AppEndpoint) with Kind
  column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`.
  Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail
  below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View,
  2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the
  Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render
  black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component
  table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The
  `decisions.authority_level = fully-automated` declaration is prominent. Most oversight fields
  are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic — this blueprint does not assume a
  human reviews every generated piece before publication; that is a deployer decision.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation`
  paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline:
  `Submit a brief. <span class="accent">Read the content.</span>`.
  Subtitle: `One agent, one guardrail.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: `Output format` dropdown (BLOG_POST / SOCIAL_POST / EMAIL_TEASER /
      PRODUCT_DESCRIPTION), `Topic` text input, `Target audience` text input, `Word count hint`
      number input (50–2000, default 300), `Submitted by` text input, and a "Load seeded brief"
      link that pre-fills all fields from one of three seeded briefs. A yellow `Generate` button
      at the bottom.
    - Live list below: one card per job, newest-first. Each card shows status pill, format badge,
      truncated topic text, and age.
  - **Right column** — Selected-job detail.
    - Header: status pill + format badge + topic text.
    - Brief section: target audience, word-count hint.
    - Generated content section (visible once `APPROVED`): the `title` in a heading style, the
      `body` in a readable block, and a word-count badge.
    - If the job is `FAILED`: a muted error message explaining that the agent could not produce
      an approved draft within the retry budget.
- Status pill colours: REQUESTED=muted, GENERATING=yellow, APPROVED=green, FAILED=red.
- Format badge colours: BLOG_POST=blue, SOCIAL_POST=purple, EMAIL_TEASER=teal,
  PRODUCT_DESCRIPTION=orange.

The raw LLM response is never displayed — only the guardrail-approved `GeneratedContent` appears.
Callers who need to inspect rejected iterations should read the service log.
