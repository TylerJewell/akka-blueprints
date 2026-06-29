# UI mockup — ghostwriter

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Ghostwriter</title>`.

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
- Headline: `Ghost<span class="accent">writer</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded voice owner (alex-chen / morgan-riley / jordan-park) and a content format.
  3. Enter a topic and click **Generate draft**.
  4. Watch the card transition through SANITIZED → DRAFTING → DRAFT_READY.
- Card **How it works**: one paragraph on submit → sanitize → draft → guardrail-check; one paragraph on the two governance mechanisms (sanitizer, after-llm-response guardrail).
- Card **Components**: rows per component (DraftEntity, CorpusSanitizer, DraftWorkflow, GhostwriterAgent, DraftOutputGuardrail, DraftView, DraftEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is prominent. `decisions.authority_level = generative-assist` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic. The `voice-owner-consent-to-ai-style-training` field is highlighted as deployer responsibility.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, G1). ID badges coloured: S1 green (sanitizer), G1 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a brief. <span class="accent">Read the draft.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Voice owner` (alex-chen / morgan-riley / jordan-park), dropdown `Format` (blog-post / memo / release-note / social), `Topic` textarea (2 rows), `Target word count` number input, optional `Additional writing sample` textarea with a `+` expand toggle, `Requested by` text input, and a yellow `Generate draft` button.
    - Live list below: one card per draft, newest-first. Each card shows status pill, format chip, fidelity-score badge (when draft landed), voice-owner label, age.
  - **Right column** — Selected-draft detail.
    - Header: status pill + format chip + fidelity-score badge + topic excerpt.
    - Writing brief summary: voice owner, format, target word count, requested by.
    - Sanitized corpus preview: one collapsible block per sample showing the redacted body and a PII-category chip row (`email`, `person-name`, etc.).
    - Draft body: a formatted prose block styled for readability.
    - Fidelity score bar: horizontal progress bar 0–100, colour-coded (green ≥ 70, yellow 40–69, red < 40).
    - Style markers: a two-column table — token name | evidence quote (italic).
    - Guardrail rejections: a small counter chip; 0 is muted, 1+ is yellow, ≥ 3 is red.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, DRAFTING=yellow, DRAFT_READY=green, FAILED=red.
- Format chip colours: blog-post=teal, memo=purple, release-note=orange, social=pink.

The raw writing samples are never displayed on this screen — only the sanitized form. Users who need the raw text fetch `/api/drafts/{id}` and read `brief.samples[].rawBody` from the JSON. This demonstrates that the model's input is the redacted form, while the audit trail keeps the originals.
