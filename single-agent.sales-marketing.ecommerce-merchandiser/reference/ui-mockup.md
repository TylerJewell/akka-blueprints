# UI mockup — merchandiser

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Merchandiser</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Merch<span class="accent">andiser</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded objective (seasonal promotion push / new-product launch / inventory clearance) and load the matching catalog scope with one click.
  3. Click **Submit objective**.
  4. Watch the card transition through CONTEXT_LOADED → GENERATING → PENDING_APPROVAL, then click Approve to publish.
- Card **How it works**: one paragraph on submit → load context → generate → approve → publish; one paragraph on the two governance mechanisms (before-tool-call guardrail, merchant approval gate).
- Card **Components**: rows per component (ProposalEntity, CatalogReader, ProposalWorkflow, MerchandiserAgent, StorefrontGuardrail, ProposalView, ProposalEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled and prominent (ecommerce catalog data is not PII). `decisions.authority_level = recommend-only`, `oversight.human_in_loop = true`, and `reviewer_must_approve_before_publish: true` are the distinctive answers. Fields marked `TO_BE_COMPLETED_BY_DEPLOYER` are faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 amber (hitl).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an objective. <span class="accent">Review the proposal.</span>`. Subtitle: `One agent, two governance gates.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Objective template` (Seasonal promotion / New product launch / Inventory clearance / Custom), `Objective` textarea (with a "Load seeded example" link that fills the objective text and scope), `Scope kind` radio (All / Category / SKU list), conditional `Category` text input or `SKU list` textarea, `Submitted by` text input, and a yellow `Submit objective` button.
    - Live list below: one card per proposal, newest-first. Each card shows status pill, confidence badge (when proposal landed), document title (first 60 chars of objective text), age.
  - **Right column** — Selected-proposal detail.
    - Header: status pill + confidence badge + first line of objective text.
    - Catalog context summary: product count, category names, active promotion count, snapshot timestamp.
    - Proposal summary: the agent's 2–4-sentence paragraph.
    - Change-recommendation table: columns type (coloured chip), target ref, current value, proposed value, rationale, confidence (coloured chip).
    - Approval controls (only visible when status is `PENDING_APPROVAL`): `Decided by` text input, `Note` textarea (optional), green **Approve** button, red **Reject** button.
    - If status is `APPROVED` or `PUBLISHED`: show the approval decision block (approver, timestamp, note).
    - If status is `REJECTED`: show the rejection decision block (who rejected, when, note).
    - If status is `EXPIRED`: show an amber expiry notice.
- Status pill colours: SUBMITTED=muted, CONTEXT_LOADED=blue, GENERATING=yellow, PENDING_APPROVAL=amber, PUBLISHING=blue, PUBLISHED=green, REJECTED=red, EXPIRED=muted-amber, FAILED=red.
- Confidence badge colours: LOW=muted, MEDIUM=blue, HIGH=green.
- ChangeType chip colours: DESCRIPTION_UPDATE=blue, PROMOTION_CREATE=green, PROMOTION_REMOVE=amber, CATEGORY_REORDER=purple, PRICE_NUDGE=yellow.
