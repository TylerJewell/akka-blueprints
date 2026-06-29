# UI mockup — post-incident-reviewer

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Post-Incident Review Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Post-Incident Review <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded incidents (or enter your own incident ID) and click **Start PIR**.
  3. Watch the card transition through GATHERING → GATHERED → ASSESSING → ASSESSED → DRAFTING → DRAFTED → AWAITING_SIGNOFF.
  4. Click **Approve** in the sign-off panel to complete the review.
- Card **How it works**: one paragraph on the three task phases (GATHER → ASSESS → DRAFT) and the typed handoff between them; one paragraph on the two governance mechanisms (before-agent-response guardrail, incident-owner sign-off).
- Card **Components**: rows per component (PIREntity, PIRWorkflow, PIRAgent, GatherTools, AssessTools, DraftTools, PIRGuardrail, PIRSignoffNotifier, PIRView, PIREndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Guardrail, and 1 Notifier as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled (incident records may contain employee names). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are displayed in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 amber (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an incident. <span class="accent">Sign off the review.</span>`. Subtitle: `One agent, three task phases, one guardrail, one human gate.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Incident ID` (with a "Pick a seeded incident" dropdown that fills it), and a yellow `Start PIR` button.
    - Live list below: one card per review, newest-first. Each card shows status pill, incident ID, severity badge (P1–P4), age, and a small red dot if any guardrail rejection fired during this review.
  - **Right column** — Selected-review detail.
    - Header: status pill + severity badge + incident title.
    - Phase panel 1 (Evidence): incident record fields + a timeline table with columns eventId, occurredAt, actor, description. Visible once `evidenceLog.isPresent()`.
    - Phase panel 2 (Impact assessment): classification fields (severity, affectedSystems, usersAffected, outageWindow) + root cause summary + contributing factors list. Visible once `impactAssessment.isPresent()`.
    - Phase panel 3 (Review): executive summary paragraph, impact classification row, filtered timeline, root cause block, action items table (description, owner, dueDate, priority chip). Visible once `review.isPresent()`.
    - Sign-off panel (only visible when `status === AWAITING_SIGNOFF`): assignee name, approve button (green), reject button (red), optional comments text area.
    - Sign-off decision row (visible after sign-off): decided by, decided at, approved/rejected badge, comments.
    - Rejection-log strip (only visible if the review has any `guardrailRejections`): a small table with check, reason, rejectedAt.
- Status pill colours: CREATED=muted, GATHERING=blue, GATHERED=blue, ASSESSING=yellow, ASSESSED=yellow, DRAFTING=blue, DRAFTED=blue, AWAITING_SIGNOFF=amber, COMPLETE=green, REJECTED=red, FAILED=red.
- Severity badge colours: P1=red, P2=orange, P3=yellow, P4=muted.

Each phase panel renders only when its data is present on the row record. A review in `ASSESSING` state shows panels 1 and 2 (panel 2 with an "in progress" spinner until the agent returns). This is the visual proof that the typed handoff between phases is the only path information travels between tasks.
