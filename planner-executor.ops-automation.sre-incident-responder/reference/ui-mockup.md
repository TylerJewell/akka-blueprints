# UI mockup — sre-incident-responder

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: SRE Incident Response Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index:

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

## Mermaid CSS overrides (Lesson 24)

The `<style>` block includes the CSS overrides AND `themeVariables` from Lesson 24 — state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these, state names in the state-machine diagram render invisible (black-on-black) and edge labels clip.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `SRE Incident Response <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit an alert in the App UI tab, watch the commander investigate and propose a remediation, approve or reject the action in the approval pane.
- Card **How it works**: one paragraph naming the components, the probe loop, the approval gate, and the post-incident eval scorer.
- Card **Components**: table with one row per component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/incidents/{id}/approve` and `/api/control/*` routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (4 agents, 1 workflow, 4 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and the placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `hitl` blue (`HT1`), `halt` red (`HT2`), `eval-event` green (`E1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an alert. <span class="accent">Watch the commander investigate.</span>` Subtitle: `The simulator drips an alert every 120 s so the page is never empty.`
- Form card: textarea labelled "Alert description", severity selector (`LOW` / `MEDIUM` / `HIGH` / `CRITICAL`), `Submit` button (yellow).
- **Approval pane** (appears per-incident when status is `AWAITING_APPROVAL`): a card showing the proposed `RemediationAction` with all fields visible (actionKind badge, target, parameters, estimatedImpact pill coloured by level), a free-text reason field, and two buttons — `Accept` (yellow) and `Reject` (muted red). On submission the pane collapses and the approval state updates via SSE.
- **Operator controls pane** (top right): shows current halt state.
  - If `halted=false`: yellow `Halt new probes` button, free-text reason field.
  - If `halted=true`: muted `HALTED` pill with reason and timestamp, plus a `Resume` button.
  - Reflects every `control-update` SSE event live.
- Live list: cards per incident; left border coloured by status (TRIAGING = muted, INVESTIGATING = blue, AWAITING_APPROVAL = yellow, REMEDIATING = orange, MITIGATED = green, UNRESOLVED = red, HALTED = orange, TIMED_OUT = pale red).
  - Header row: description (first 80 chars), status pill, probe attempt count, elapsed time.
  - Click to expand:
    - **Investigation ledger**: facts (yellow list), hypotheses (muted list), probe plan (numbered list), currentProbe (probeKind pill + target line).
    - **Probe log**: vertical timeline of `ProbeEntry` rows. Each row shows attempt number, probeKind pill (`METRICS` blue, `LOGS` green, `TRACES` purple, `RUNBOOK` muted), target, verdict pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `FAILED` red, `UNSAFE` dark red), and a collapsed `<pre>` of `result` (click to expand).
    - **Remediation ledger** (when populated): each proposed action as a card with approval status badge. Approved actions show the `ActionOutcome` below.
    - **Post-incident report** (only when `MITIGATED` or `UNRESOLVED`): summary paragraph, root-cause block, timeline list, lessons-learned bullets, and optional follow-up actions.
    - **Eval score** (when present): two circular score badges (`investigationCoverageScore` and `hypothesisAccuracyScore` on a 1–10 scale, coloured green ≥ 7, amber 4–6, red ≤ 3), time-to-mitigate stat, and findings list.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
