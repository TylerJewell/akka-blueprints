# UI mockup — screenplay-writer-marketplace

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Screenplay Writer Team</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index (Lesson 26):

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Screenplay <span class="accent">Writer Team</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded sources (or paste your own email thread) and click **Write screenplay**.
  3. Watch the card transition through PARSING → PARSED → DEVELOPING → DEVELOPED → FORMATTING → FORMATTED → DELIVERED.
  4. Inspect the delivery-block strip on the card if the guardrail blocked delivery.
- Card **How it works**: one paragraph on the three task phases (PARSE → DEVELOP → FORMAT) and the typed handoff between them; one paragraph on the two governance mechanisms (PII sanitizer at ingestion, delivery guardrail at egress).
- Card **Components**: rows per component (ScreenplayEntity, ScreenplayPipelineWorkflow, ScreenplayAgent, ParseTools, DevelopTools, FormatTools, PiiSanitizer, DeliveryGuardrail, ScreenplayView, ScreenplayEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Sanitizer, and 1 Guardrail as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled (source emails carry real names and addresses). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Deployer fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 teal (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Paste an email thread. <span class="accent">Read the screenplay.</span>`. Subtitle: `One agent, three task phases, PII scrubbed at ingestion and verified at delivery.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Source title`, textarea `Source text` (with a "Pick a seeded source" dropdown that fills both), and a yellow `Write screenplay` button.
    - Live list below: one card per screenplay, newest-first. Each card shows status pill, delivery-check chip (PASSED/BLOCKED when landed), source title, age, and a small amber dot if the delivery guardrail blocked this screenplay.
  - **Right column** — Selected-screenplay detail.
    - Header: status pill + delivery-check chip + source title.
    - Phase panel 1 (Parsed source): a table with columns character placeholder, archetype; a table with settings; a table with beats and dramatic function. Visible once `parsedSource` is present.
    - Phase panel 2 (Scene plan): a scene list with sluglines and assigned beat/character ids. Visible once `scenePlan` is present.
    - Phase panel 3 (Screenplay): title, logline, then per-scene block (slugline, action paragraph, dialogue lines). Visible once `screenplay` is present.
    - Delivery-check section at bottom: a PASSED (green) or BLOCKED (red) badge and, if blocked, the list of detected tokens.
    - Delivery-block log strip (only visible if `deliveryCheck.passed == false`): a small table with detected token, reason, and time.
- Status pill colours: CREATED=muted, PARSING=blue, PARSED=blue, DEVELOPING=yellow, DEVELOPED=yellow, FORMATTING=blue, FORMATTED=blue, DELIVERED=green, DELIVERY_BLOCKED=red, FAILED=red.

Each phase panel renders only when its data is present on the row record. A screenplay in `DEVELOPING` shows panel 1 (present) but not panel 2 (not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
