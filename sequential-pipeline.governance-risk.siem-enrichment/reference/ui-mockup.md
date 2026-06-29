# UI mockup — siem-enrichment

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SIEM Alert Enrichment</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `SIEM Alert <span class="accent">Enrichment</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded alerts (or paste a raw alert JSON string) and click **Run pipeline**.
  3. Watch the card transition through FETCHING → FETCHED → ENRICHING → ENRICHED → TRIAGING → TRIAGED → EVALUATED.
  4. Inspect the rejection-log strip on the card if any scope-gate rejections fired.
- Card **How it works**: one paragraph on the three task phases (FETCH → ENRICH → TRIAGE) and the typed handoff between them; one paragraph on the two governance mechanisms (ticket-scope guardrail, triage quality eval).
- Card **Components**: rows per component (AlertEntity, AlertEnrichmentWorkflow, AlertEnrichmentAgent, FetchTools, EnrichTools, TicketTools, TicketScopeGuardrail, TriageQualityScorer, AlertView, AlertEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled (SIEM alerts contain IP addresses and hostnames; this distinguishes the governance-risk domain from a general-purpose pipeline). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, E1). ID badges coloured: G1 red (guardrail), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an alert. <span class="accent">Read the ticket.</span>`. Subtitle: `One agent, three task phases, one runtime gate before the Zendesk write.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a "Pick a seeded alert" dropdown that fills a text area with the raw alert JSON, and a red `Run pipeline` button.
    - Live list below: one card per alert, newest-first. Each card shows status pill, eval score chip (when eval landed), rule name, age, and a small red dot if any guardrail rejection fired during this alert.
  - **Right column** — Selected-alert detail.
    - Header: status pill + eval score chip + rule name.
    - Phase panel 1 (Alert detail): a table of raw SIEM fields (key → value). Visible once `alertDetail.isPresent()`.
    - Phase panel 2 (Enrichment): an attack-pattern table (techniqueId, techniqueName, tactic, confidence) and derived severity badge. Visible once `enrichedAlert.isPresent()`.
    - Phase panel 3 (Triage ticket): title, severity, assigned team, Zendesk ticket link, summary paragraph. Visible once `triageTicket.isPresent()`.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Rejection-log strip (only visible if the alert has any `guardrailRejections`): a small table with phase, tool, reason, time.
- Status pill colours: RECEIVED=muted, FETCHING=blue, FETCHED=blue, ENRICHING=yellow, ENRICHED=yellow, TRIAGING=blue, TRIAGED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. An alert in `ENRICHING` shows panel 1 and the beginning of panel 2 (panel 2 with a "in progress" spinner if the agent has not yet returned). This is the visual proof that the typed handoff between phases is the only path information travels.
