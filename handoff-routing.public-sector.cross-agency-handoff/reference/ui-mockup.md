# UI mockup — cross-agency-case-handoff-mesh

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Cross-Agency Case Handoff Mesh</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Cross-Agency Case <span class="accent">Handoff Mesh</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block, then four numbered steps (open App UI, wait 30 s for the simulator to drop the first case, click any case to inspect, click Approve or Reject on a case in `HANDOFF_PENDING`).
- Card **How it works**: one paragraph on the scope → route → jurisdiction check → agency segment → HITL approval → re-emit or close flow; one paragraph on the three governance mechanisms.
- Card **Components**: rows per component (simulator, inbox, scoping filter, jurisdiction router, intake assessor, benefits reviewer, jurisdiction guardrail, workflow, entity, view, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents typed, 2 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs styled from `governance.html`. The distinctive declarations are `data.pii = true`, `data.pii_scoped_per_agency_before_llm = true`, `decisions.authority_level = human-approved-handoff`, `oversight.human_in_loop = true`, and `oversight.reviewer_must_approve_every_handoff = true`. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Three rows: S1 (sanitizer · cyan badge), H1 (hitl · amber badge), G1 (guardrail · yellow badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the handoff. <span class="accent">Every transfer needs a signature.</span>`. Subtitle: `Simulated cases drop every 30 s. Route, jurisdiction check, agent assessment, then an operator approves or rejects each cross-agency transfer.`
- Layout: three-column.
  - **Left column** — Live case list, sorted newest-first. Each card shows:
    - Header: status pill, segment badge (INTAKE amber, BENEFITS cyan, UNROUTABLE muted), program-type chip, case age.
    - Redacted summary (the raw summary is never shown).
    - Dropped field categories (small muted chips, e.g. "ssn", "dob", "income").
  - **Centre column** — Selected case: redacted summary + payload (read-only), the routing block (segment badge + confidence + rationale), and the jurisdiction verdict block (green check when allowed; red badge + violations list when blocked).
  - **Right column** — The active segment's outcome:
    - Agency tag (intake amber, benefits cyan).
    - Determination chip (`ELIGIBLE`, `AWARD_ISSUED`, `PENDING_DOCUMENTS`, etc.).
    - Summary note (the automated draft).
    - Approval panel when `status = HANDOFF_PENDING`:
      - Two buttons: **Approve** (green) and **Reject** (red).
      - Approve opens a small note textarea and posts to `/api/cases/{id}/approve-handoff`.
      - Reject opens a reason textarea and posts to `/api/cases/{id}/reject-handoff`.
    - When `status = HANDOFF_EMITTED`: the approval record (approvedBy + note + timestamp) with a green stamp.
    - When `status = CLOSED`: a green "Case Closed" block with the final segment's outcome and `closedAt`.
    - When `status = HANDOFF_REJECTED`: a red "Transfer Rejected" block with the rejection reason and the rejected determination (still visible for audit).
    - When `status = JURISDICTION_BLOCKED`: a red "Jurisdiction Blocked" block with the violations list; no segment outcome.
    - When `status = UNROUTABLE`: a muted "Unroutable — no agency segment" block with the routing rationale.
- Status pill colours: `RECEIVED` / `SCOPED` muted, `ROUTED` blue, `JURISDICTION_CHECK` yellow, `IN_SEGMENT` purple, `SEGMENT_COMPLETE` teal, `HANDOFF_PENDING` amber (pulsing), `HANDOFF_EMITTED` cyan, `HANDOFF_REJECTED` red, `JURISDICTION_BLOCKED` red, `CLOSED` green, `UNROUTABLE` muted-red.

The `HANDOFF_PENDING` amber-pulsing pill is the most operationally distinctive surface — it immediately signals to the case-officer which cases need their attention. The Approve / Reject buttons are the HITL mechanism made tangible.
