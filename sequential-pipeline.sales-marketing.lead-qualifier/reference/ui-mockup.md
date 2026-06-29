# UI mockup — lead-qualifier

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Lead Qualifier</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Lead <span class="accent">Qualifier</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded inquiries (or paste your own) and click **Qualify lead**.
  3. Watch the card transition through CAPTURING → CAPTURED → QUALIFYING → QUALIFIED → ENRICHING → ENRICHED → EVALUATED.
  4. Inspect the rejection-log strip on the card if any guardrail rejections fired.
- Card **How it works**: one paragraph on the three task phases (CAPTURE → QUALIFY → ENRICH) and the typed handoff between them; one paragraph on the two governance mechanisms (CRM write + phase-gate guardrail, PII masking sanitizer).
- Card **Components**: rows per component (LeadEntity, LeadPipelineWorkflow, InquiryAgent, CaptureTools, QualifyTools, EnrichTools, CrmWriteGuardrail, PiiSanitizer, QualityScorer, LeadView, LeadEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, 1 Sanitizer, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled — lead capture includes personal contact data. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 purple (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Paste an inquiry. <span class="accent">Get a qualified lead.</span>`. Subtitle: `One agent, three task phases, one gate between the pipeline and CRM.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: textarea `Inquiry text` (with a "Pick a seeded inquiry" dropdown that fills it), and a yellow `Qualify lead` button.
    - Live list below: one card per lead, newest-first. Each card shows status pill, data-quality score chip (when eval landed), company name (or "—" if not yet captured), age, and a small red dot if any guardrail rejection fired during this lead.
  - **Right column** — Selected-lead detail.
    - Header: status pill + data-quality score chip + company name.
    - Phase panel 1 (Captured form): a table with columns field, value — name-initial, company, channel, productInterest, budgetIndicator, capturedAt. Visible once `form.isPresent()`. Raw email and phone are never shown.
    - Phase panel 2 (Lead score): fit score bar (0–100), urgency badge, recommended stage chip, disqualification reason if present. Visible once `score.isPresent()`.
    - Phase panel 3 (CRM entry): stage chip, owner candidate, notes text, masked email, masked phone, name initial, enrichedAt. Visible once `crmEntry.isPresent()`.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the lead has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: CREATED=muted, CAPTURING=blue, CAPTURED=blue, QUALIFYING=yellow, QUALIFIED=yellow, ENRICHING=blue, ENRICHED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A lead in `QUALIFYING` shows panel 1 (the captured form) and panel 2 with a spinner if the agent has not yet returned the score. This is the visual proof that the typed handoff between phases is the only path information travels.
