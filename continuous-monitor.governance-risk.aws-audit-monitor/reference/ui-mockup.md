# UI mockup — aws-audit-monitor

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: AWS Audit Monitor</title>`.

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

And the DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `AWS Audit <span class="accent">Monitor</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the scanner to drop a finding, review the compiled report, click Publish or Dismiss).
- Card **How it works**: one paragraph on the scan → normalize → analyze → compile → review flow; one paragraph on the two governance mechanisms (HITL publish gate, periodic accuracy eval).
- Card **Components**: rows per component (poller, queue, normalizer, analyst, compiler, judge, workflow, finding entity, report entity, view, evalrunner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents, 1 AutonomousAgent, 1 Workflow, 3 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `cloud-account-identifiers: true` and `account_ids_normalized_before_llm: true` declarations in Data are the most distinctive. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (H1, E1). ID badges coloured: H1 pink (hitl), E1 green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the findings. <span class="accent">Publish every report.</span>`. Subtitle: `Simulated AWS findings drop every 60 s. Reports are held until a risk officer acts.`
- Layout: two-column.
  - **Left column** — Split into two stacked panels:
    - *Findings feed* (top): Finding cards sorted newest-first. Each card shows:
      - Header: status pill, severity chip (CRITICAL=red, HIGH=orange, MEDIUM=yellow, LOW=blue, INFO=muted), finding age.
      - Normalized resource type and ARN (masked).
      - Control domain chip (identity/data-protection/network/logging/encryption).
    - *Reports panel* (bottom): Report cards sorted newest-first. Each shows: status pill, finding count by severity, report age, Publish/Dismiss controls.
  - **Right column** — the selected item detail.
    - If a finding is selected: normalized detail (read-only monospace), analysis output (severity, controlRef, remediationSummary), eval score chip (when populated).
    - If a report is selected: executive summary (read-only), prioritized finding list with severity chips, Publish button (green), Dismiss button (border), Dismiss opens a "Reason" textarea.
    - After action, the decision metadata (reviewedBy, decidedAt) is visible.
- Status pill colours: DETECTED=muted, NORMALIZED=muted, ANALYZED=blue, COMPILED=blue, PENDING_REVIEW=yellow, PUBLISHED=green, DISMISSED=muted.

Publishing a report transitions all linked findings simultaneously. The finding cards re-render to PUBLISHED without a page reload.
