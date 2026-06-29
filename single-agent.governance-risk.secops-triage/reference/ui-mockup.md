# UI mockup — secops-triage

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: SecOps Vulnerability Triage Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `SecOps <span class="accent">Vulnerability Triage</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded finding (container escape / SQL injection / misconfiguration) and load it with one click.
  3. Click **Ingest finding**.
  4. Watch the card transition through ENRICHED → TRIAGING → VERDICT_RECORDED → PENDING_APPROVAL (for critical/high) and approve or reject.
- Card **How it works**: one paragraph on ingest → enrich → triage → approval-gate; one paragraph on the three governance mechanisms (before-tool-call guardrail, analyst approval gate, periodic drift evaluator).
- Card **Components**: rows per component (FindingEntity, ApprovalEntity, FindingEnricher, TriageWorkflow, DriftCheckWorkflow, VulnerabilityTriageAgent, RemediationGuardrail, DriftEvaluator, FindingView, FindingEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 2 Workflows + 1 DriftCheckWorkflow, 2 EventSourcedEntities, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `security-finding-data: true` and `asset-configuration-data: true` declarations in Data are filled and prominent. `decisions.authority_level = recommend-and-gate` and `oversight.human_in_loop = true` are the distinctive answers. `high_critical_decisions_require_approval: true` is highlighted. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, H1, E1). ID badges coloured: G1 red (guardrail), H1 orange (hitl), E1 blue (eval-periodic).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ingest a finding. <span class="accent">Approve the fix.</span>`. Subtitle: `One agent, three governance mechanisms around it.`
- **Drift alert banner** (conditional): if a `drift-alert` SSE event has been received, a yellow banner at the top of the panel shows the alert description and the baseline vs. observed percentages.
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Load seeded finding` (container-escape / sql-injection / misconfiguration), `CVE ID` text input, `CVSS Score` number input, `Affected Asset` text input, `Description` textarea, `Submitted by` text input, and a yellow `Ingest finding` button.
    - Live list below: one card per finding, newest-first. Each card shows priority badge (when verdict landed), status pill, CVE id, age. Findings in `PENDING_APPROVAL` show an amber pulsing ring.
  - **Right column** — Selected-finding detail.
    - Header: priority badge + status pill + CVE id + affected asset.
    - Raw finding: CVE id, CVSS score chip (red ≥ 9.0, orange ≥ 7.0, blue ≥ 4.0, grey < 4.0), description.
    - Enriched context: asset criticality tier badge, internet-facing flag, existing mitigations, owner team, threat-intel summary, exploit-in-wild chip.
    - Triage verdict: priority badge, risk rationale paragraph, recommended action.
    - Approval panel (visible only when `PENDING_APPROVAL`): approve button (green), reject button (red), analyst-id input, reason textarea.
    - Approval result (visible after `REMEDIATED` or `REMEDIATION_REJECTED`): analyst id, decision badge, reason, decided-at timestamp.
- Status pill colours: INGESTED=muted, ENRICHED=blue, TRIAGING=yellow, VERDICT_RECORDED=blue, PENDING_APPROVAL=amber, REMEDIATED=green, REMEDIATION_REJECTED=red, MONITORED=teal, ACCEPTED=grey, FAILED=red.
- Priority badge colours: CRITICAL_IMMEDIATE=red, HIGH_SCHEDULED=orange, MEDIUM_MONITORED=blue, LOW_ACCEPTED=grey.
