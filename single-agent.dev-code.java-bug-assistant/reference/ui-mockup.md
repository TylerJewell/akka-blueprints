# UI mockup — java-bug-assistant

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Software Bug Assistant</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Software Bug <span class="accent">Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded bug category (runtime-error / network-failure / resource-exhaustion) and load the matching seed report with one click.
  3. Click **Submit bug**.
  4. Watch the card transition through NORMALIZED → SEARCHING → SEARCH_COMPLETE → RESOLVING → RECOMMENDATION_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on submit → normalize → search → resolve → eval; one paragraph on the governance mechanism (before-tool-call guardrail).
- Card **Components**: rows per component (BugEntity, BugNormalizer, TicketSearcher, TicketStore, ResolutionWorkflow, BugResolutionAgent, TicketWriteGuardrail, RecommendationScorer, BugView, BugEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 Consumers, 1 TicketStore, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `internal-credentials: true` and `internal-hostnames: true` declarations in Data are filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Deployer fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a bug. <span class="accent">Read the recommendation.</span>`. Subtitle: `One agent, one guardrail around every ticket write.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Bug category` (runtime-error / network-failure / resource-exhaustion / auto-detect), `Title` text input, `Bug report` textarea (with a "Load seeded example" link that fills both title and body), `Submitted by` text input, and a yellow `Submit bug` button.
    - Live list below: one card per bug, newest-first. Each card shows status pill, action badge (when recommendation landed), eval score chip (when eval landed), bug title, age.
  - **Right column** — Selected-bug detail.
    - Header: status pill + action badge + eval score chip + `title`.
    - Normalized report preview: a monospace block of the cleaned report, with redaction category chips above (`credential`, `internal-hostname`, `stack-trace-token`).
    - Related tickets panel: a small table of matched tickets with `ticketId`, `ticketStatus` chip, and `summary`. Resolution text shown inline for RESOLVED/CLOSED tickets.
    - Recommendation rationale: the agent's 2–4-sentence paragraph.
    - Candidate-fix ranked list: columns fix id, confidence bar (0–100), description, source ref (linked).
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: SUBMITTED=muted, NORMALIZED=blue, SEARCHING=yellow, SEARCH_COMPLETE=blue, RESOLVING=yellow, RECOMMENDATION_RECORDED=blue, EVALUATED=green, FAILED=red.
- Action badge colours: RESOLVED=green, NEEDS_INVESTIGATION=yellow, ESCALATE=red.
- Confidence bar colours: 0–29=red, 30–59=yellow, 60–100=green.

The raw bug report is never displayed on this screen — only the normalized form. Engineers who need the raw text fetch `/api/bugs/{id}` and read `report.rawReport` from the JSON. This demonstrates that the model's input is the cleaned form, even though the audit trail retains the raw submission.
