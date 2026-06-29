# UI mockup — inbound-lead-qualification

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Inbound Lead Qualification & Research Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The canonical implementation:

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
- Headline: `Inbound Lead Qualification <span class="accent">&amp; Research Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Fill in the Submit Lead form (or pick one of the seeded leads) and click **Submit lead**.
  3. Watch the card transition through ENRICHING → ENRICHED → QUALIFYING → QUALIFIED → NOTIFYING → NOTIFIED → EVALUATED.
  4. Inspect the rejection-log strip on the card if any guardrail rejections fired.
- Card **How it works**: one paragraph on the three task phases (ENRICH → QUALIFY → NOTIFY) and the typed handoff between them; one paragraph on the three governance mechanisms (PII sanitizer, Slack write guardrail, qualification accuracy eval).
- Card **Components**: rows per component (LeadEntity, LeadPipelineWorkflow, LeadAgent, EnrichTools, QualifyTools, NotifyTools, PiiSanitizer, SlackGuardrail, QualificationEvaluator, LeadView, LeadEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, 1 Sanitizer, and 1 Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled (the lead form carries name and email). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, S1, E1). ID badges coloured: G1 red (guardrail), S1 violet (sanitizer), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a lead. <span class="accent">Score it. Notify the rep.</span>`. Subtitle: `One agent, three task phases, PII stripped before enrichment begins.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: fields for first name, last name, email, company name, company size (dropdown: XS / S / M / L / XL), optional message. A "Pick a seeded lead" dropdown fills all fields. Yellow `Submit lead` button.
    - Live list below: one card per lead, newest-first. Each card shows status pill, tier chip (HOT=red, WARM=yellow, COLD=blue — visible once qualified), confidence chip (HIGH/MED/LOW — visible once evaluated), company name, age, and a small red dot if any guardrail rejection fired.
  - **Right column** — Selected-lead detail.
    - Header: status pill + tier chip + confidence chip + company name.
    - Phase panel 1 (Enrichment): a table with columns domain, industry, estimated ARR, employee band, tech stack signals. Visible once `profile.isPresent()`.
    - Phase panel 2 (Qualification): tier badge, score bar (0–100), rationale text, assigned rep name. Visible once `score.isPresent()`.
    - Phase panel 3 (Slack notification): channel, message text, blocks preview (rendered as a minimal Slack-style card). Visible once `notification.isPresent()`.
    - Eval section at bottom: confidence chip (HIGH green / MED yellow / LOW amber), one-line rationale. LOW confidence highlights the card border amber.
    - Rejection-log strip (only visible if the lead has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: SUBMITTED=muted, ENRICHING=blue, ENRICHED=blue, QUALIFYING=yellow, QUALIFIED=yellow, NOTIFYING=blue, NOTIFIED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A lead in `QUALIFYING` shows panels 1 and 2 (panel 2 with a "qualifying…" spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels between ENRICH and QUALIFY.
