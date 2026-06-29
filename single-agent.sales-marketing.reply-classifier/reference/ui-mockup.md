# UI mockup — reply-classifier

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: ReplyClassifier</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (see Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Reply<span class="accent">Classifier</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded reply (Interested / Objection / Out of Office / Unsubscribe) or paste a raw email reply.
  3. Enter a Deal ID and click **Classify reply**.
  4. Watch the card transition through CLASSIFYING → CLASSIFIED → CRM_UPDATED (or CRM_SKIPPED).
- Card **How it works**: one paragraph on submit → classify → CRM update; one paragraph on the guardrail and what it prevents.
- Card **Components**: rows per component (ReplyEntity, ClassificationWorkflow, ReplyClassifierAgent, CrmMutationGuardrail, PipedriveClient, ReplyView, ReplyEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, 1 Guardrail + 1 Client as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent with a note that no pre-LLM redaction is applied in this baseline. `decisions.authority_level = automated-with-guardrail` and `oversight.human_on_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Classify a reply. <span class="accent">Update the deal.</span>`. Subtitle: `One agent, one guardrail around every CRM mutation.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded reply` (Interested / Price Objection / Out of Office / Unsubscribe / custom), `Deal ID` text input, `Sender` text input, `Subject` text input, `Reply text` textarea (with a "Load seeded example" link that fills all fields), and a yellow `Classify reply` button.
    - Live list below: one card per reply, newest-first. Each card shows status pill, intent badge (when classified), confidence chip, deal id, sender (shortened), age.
  - **Right column** — Selected-reply detail.
    - Header: status pill + intent badge + confidence score + deal id.
    - Raw reply text: a monospace block of the submitted reply text.
    - Classification: intent badge, confidence progress bar (0–100), rationale paragraph.
    - CRM action: action type chip (`UPDATE_STAGE` or `SKIP_CRM_UPDATE`), proposed stage label, and — after `crmUpdateStep` completes — the outcome (`CRM_UPDATED` with previous→new stage, or `CRM_SKIPPED` with skip reason).
    - Guardrail event (when present): a yellow banner showing the rejection reason and iteration count, so the user can see when the guardrail fired.
- Status pill colours: RECEIVED=muted, CLASSIFYING=yellow, CLASSIFIED=blue, CRM_UPDATED=green, CRM_SKIPPED=blue, FAILED=red.
- Intent badge colours: INTERESTED=green, NOT_INTERESTED=muted, OBJECTION=yellow, OUT_OF_OFFICE=blue, UNSUBSCRIBE=red.
- Action type chip: UPDATE_STAGE=green, SKIP_CRM_UPDATE=muted.
