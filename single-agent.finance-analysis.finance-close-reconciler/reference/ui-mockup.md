# UI mockup — finance-close-reconciler

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Finance Close Reconciler</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Finance Close <span class="accent">Reconciler</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded rule set (IFRS-operating / GAAP-manufacturing / interco-eliminations) and load the matching seed balance, or paste a custom trial balance.
  3. Click **Submit for reconciliation**.
  4. Watch the card transition through SANITIZED → RECONCILING → REPORT_RECORDED → AWAITING_SIGNOFF, then click **Approve** to reach ATTESTED.
- Card **How it works**: one paragraph on submit → sanitize → reconcile → sign-off → attest; one paragraph on the four governance mechanisms (GL-write guardrail, accountant sign-off, confidential-field sanitizer, CI attestation gate).
- Card **Components**: rows per component (PeriodEntity, BalanceSanitizer, ReconciliationWorkflow, ReconciliationAgent, WriteGuardrail, AttestationScorer, ReconciliationView, ReconciliationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `financial-records: true` and `confidential-margin-data: true` declarations in Data are filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with four rows (G1, H1, S1, CI1). ID badges coloured: G1 red (guardrail), H1 orange (hitl), S1 green (sanitizer), CI1 purple (ci-gate).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a balance. <span class="accent">Reconcile the period.</span>`. Subtitle: `One agent, four governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Rule set` (IFRS-operating / GAAP-manufacturing / interco-eliminations / custom), `Period label` text input, `Trial Balance (CSV)` textarea (with a "Load seeded example" link that fills both label and balance), `Submitted by` text input, and a yellow `Submit for reconciliation` button.
    - Live list below: one card per period, newest-first. Each card shows status pill, report badge (when report landed), attestation chip (when attested), period label, age.
  - **Right column** — Selected-period detail.
    - Header: status pill + report badge + attestation chip + `periodLabel`.
    - Submitted rules: a small table with `ruleId`, `accountPair`, `toleranceAmount`, `material` flag chip, and `description`.
    - Masked balance preview: a monospace block of the masked CSV, with masked-field-type chips above (`margin-pct`, `entity-code`, etc.).
    - Report summary: the agent's 1–3-sentence paragraph.
    - Findings table: columns rule id, account pair, finding (coloured chip), expected balance, actual balance, variance, note.
    - Proposed GL writes: a small list of the write proposals that passed the guardrail.
    - Sign-off section: when status is AWAITING_SIGNOFF, shows an **Approve** button and a **Reject** button with a note textarea. When status is ATTESTED, shows the attestation chip. When REJECTED, shows the rejection note in muted text.
    - Attestation section at bottom: complete flag, attested-by, attested-at.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, RECONCILING=yellow, REPORT_RECORDED=blue, AWAITING_SIGNOFF=orange, ATTESTED=green, REJECTED=muted-red, FAILED=red.
- Report badge colours: IN_BALANCE=green, HAS_VARIANCES=yellow, INCOMPLETE=red.
- Finding chip colours: IN_BALANCE=green, VARIANCE=yellow, UNMATCHED=red.

The raw trial balance is never displayed on this screen — only the masked form. Accountants who need the raw balance fetch `GET /api/periods/{id}` and read `submission.rawBalance` from the JSON. This is intentional: the UI demonstrates that the model's input is the masked form, even though the audit trail retains the original.
