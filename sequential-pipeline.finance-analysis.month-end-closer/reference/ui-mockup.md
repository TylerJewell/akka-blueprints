# UI mockup — month-end-closer

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Month-End Closer</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Month-End <span class="accent">Closer</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick an entity and period from the dropdowns and click **Start close run**.
  3. When the card reaches `AWAITING_GATHER_APPROVAL`, review the ledger snapshot and click **Approve gather**.
  4. When the card reaches `AWAITING_VALIDATE_APPROVAL`, review the journal entries and click **Approve validate**. The run will complete to `EVALUATED`.
- Card **How it works**: one paragraph on the three task phases (GATHER → VALIDATE → REPORT) and the typed handoff between them; one paragraph on the two governance mechanisms (HITL approval gate after each of the first two phases, reconciliation eval on the close package).
- Card **Components**: rows per component (CloseRunEntity, CloseRunWorkflow, CloseAgent, GatherTools, ValidateTools, ReportTools, ApprovalGate, ReconciliationScorer, CloseRunView, CloseRunEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 ApprovalGate helper, and 1 ReconciliationScorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (ledger lines are financial amounts and account codes, not person-level data). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` with two explicit approval gates are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, E1). ID badges coloured: H1 amber (hitl), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Start a close run. <span class="accent">Two approvals. One package.</span>`. Subtitle: `One agent, three task phases, two human gates between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: entity dropdown (ACME-US, ACME-EU, WIDGET-US or custom text), period dropdown (last 3 months pre-populated), and a yellow **Start close run** button.
    - Live list below: one card per close run, newest-first. Each card shows status pill, reconciliation score chip (when eval landed), entity + period, age, and a small amber dot if the run is awaiting approval.
  - **Right column** — Selected-close-run detail.
    - Header: status pill + reconciliation score chip + entity + period.
    - Phase panel 1 (Ledger snapshot): a table with columns account code, description, debit amount, credit amount, posting date. Visible once `ledgerSnapshot.isPresent()`. When status is `AWAITING_GATHER_APPROVAL`, an **Approve gather** / **Reject gather** button pair appears below the table.
    - Phase panel 2 (Journal entries): a table with columns entry id, debit account, credit account, amount, narration, reversal date. Visible once `journalEntrySet.isPresent()`. When status is `AWAITING_VALIDATE_APPROVAL`, an **Approve validate** / **Reject validate** button pair appears below the table.
    - Phase panel 3 (Close package): title, trial balance summary row (total debits, total credits, variance, balanced badge), journal entries list, variance commentary paragraph. Visible once `closePackage.isPresent()`.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Approval history strip: a small table with step, approver, decision (APPROVED / REJECTED), comment, timestamp. Visible for any close run with at least one approval or rejection event.
- Status pill colours: CREATED=muted, GATHERING=blue, AWAITING_GATHER_APPROVAL=amber, VALIDATING=yellow, AWAITING_VALIDATE_APPROVAL=amber, REPORTING=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A close run in `VALIDATING` shows panels 1 and 2 (panel 2 with an "in progress" indicator if the agent has not yet returned). The approval button pair appears only in the two `AWAITING_*_APPROVAL` states — never otherwise.
