# UI mockup — gl-reconciler

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: General Ledger Reconciler</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `General Ledger <span class="accent">Reconciler</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded account sets (or type your own) and click **Run reconciliation**.
  3. Watch the card transition through FETCHING → FETCHED → RECONCILING → RECONCILED → DRAFTING → DRAFT_VALIDATED → POSTED. If a material variance is detected, the card pauses at PENDING_APPROVAL — approve or reject from the card detail.
  4. Inspect the validation-failure strip if the posting guardrail rejected a draft.
- Card **How it works**: one paragraph on the three task phases (FETCH → RECONCILE → DRAFT) and the typed handoff between them; one paragraph on the two governance mechanisms (posting-safety guardrail, material-variance HITL escalation).
- Card **Components**: rows per component (LedgerReconciliationEntity, ReconciliationWorkflow, LedgerAgent, FetchTools, ReconcileTools, JournalTools, PostingGuardrail, MaterialVarianceChecker, ReconciliationView, ReconciliationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Checker as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `financial-account-data: true` declaration in Data is filled (the ledger entries are financial data). `decisions.authority_level = operational-write` and `oversight.human_in_loop = true` with `controller_must_approve_before_post_on_material_variance: true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 amber (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an account set. <span class="accent">Post the journal.</span>`. Subtitle: `One agent, three task phases, one controller gate for material variances.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Account set` (with a "Pick a seeded account set" dropdown that fills it), and a yellow `Run reconciliation` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, account set identifier, age, and a small amber dot if the run is waiting for controller approval.
  - **Right column** — Selected-run detail.
    - Header: status pill + account set identifier.
    - Phase panel 1 (Ledger entries): a table with columns account ID, description, debit, credit, date. Visible once `snapshot.isPresent()`.
    - Phase panel 2 (Reconciliation): a variance table (account, expected balance, actual balance, delta, material flag) and a NAV line (total assets, total liabilities, NAV). Visible once `reconciliation.isPresent()`.
    - **Escalation strip** (only visible when `status == PENDING_APPROVAL`): the escalation reason, an `Approve` button, and a `Reject` button with a reason text field. Submits to `POST /api/reconciliations/{id}/approve` or `/reject`.
    - Phase panel 3 (Journal entry): journal-lines table (account ID, debit, credit, description), total-debits chip, total-credits chip, balance-check chip (green ✓ balanced / red ✗ unbalanced — always ✓ by the time the panel is visible, since the guardrail blocked any unbalanced draft). Visible once `journal.isPresent()`.
    - Validation-failure strip (only visible if `validation.findings` is non-empty): a small table with the finding text per failed rule.
- Status pill colours: CREATED=muted, FETCHING=blue, FETCHED=blue, RECONCILING=yellow, RECONCILED=yellow, PENDING_APPROVAL=amber, DRAFTING=blue, DRAFT_VALIDATED=blue, POSTED=green, REJECTED=muted, FAILED=red.

Each phase panel renders only when its data is present on the row record. A run in `RECONCILING` shows panel 1 and the start of panel 2 (with a spinner if the agent has not yet returned). The escalation strip is visible only in `PENDING_APPROVAL` — it disappears once the controller decides. This layout makes the HITL path impossible to miss.
