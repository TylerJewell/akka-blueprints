# UI mockup — ad-spend-payment-agent

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: AI Ads Generation Agent with Crypto Payments</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index.

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI. Any panel removed in a prior iteration must be deleted, not hidden with `display:none`. See Lesson 26.

## Mermaid CSS overrides (Lesson 24)

The `<style>` block includes the CSS overrides AND `themeVariables` from Lesson 24: state-diagram label colour, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`. Without these the state-machine diagram renders state names invisible and `AWAITING_APPROVAL` label clips.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `AI Ads Generation Agent <span class="accent">with Crypto Payments</span>`. **No subtitle.**
- Card **Try it**: a single code block reading `/akka:build`. Below it, three numbered steps — submit a campaign brief in the App UI tab, approve the placement spend in the pending-approvals pane, expand the campaign row to see both ledgers and the final report.
- Card **How it works**: one paragraph naming the components, the loop steps (plan → copywriter → approval gate → payment), and the three governance controls.
- Card **Components**: table with one row per component listed in SPEC §4. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`, including the `/api/campaigns/{id}/approvals/*` and `/api/control/*` routes.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (3 agents, 1 workflow, 4 entities, 1 view, 1 consumer, 2 timed-actions, 2 endpoints).
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sub-tabs in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each `.qb` rendered with the question text, drives sublabel, and the chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Unanswered `.qb` blocks get `opacity: 0.45` and placeholder text "To be completed by deployer".

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges coloured per mechanism: `guardrail` red (`G1`), `hitl` blue (`HI1`), `halt` orange (`HO1`).
- Rows expand vertically on click; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a brief. <span class="accent">Watch the loop run.</span>` Subtitle: `The simulator drips a campaign every 90 s so the page is never empty.`
- Form card: fields for Title, Brand Voice (text), Target Audience (text), Ad Format (select: banner-728x90, leaderboard-728x90, square-300x250, vertical-160x600), Budget in ETH (number, converted to wei internally), `Submit` button (yellow).
- **Pending-approvals pane** (below the form): a card listing all placement approvals with status `PENDING`. Each row shows the campaign title, placement task, amount (formatted in ETH), destination wallet (truncated to first 8 + last 4 chars), and two buttons — `Approve` (yellow) and `Reject` (muted red with a reason field). Clicking either calls the relevant approval endpoint and updates the pane live via `approval-update` SSE events.
- **Operator controls pane** (top right of the App UI tab):
  - If `halted=false`: yellow `Halt new dispatches` button, free-text reason field.
  - If `halted=true`: muted `HALTED` pill with the reason and timestamp, plus a `Resume` button.
  - The pane reflects every `control-update` SSE event live.
- Live list: cards per campaign; left border coloured by status (PLANNING = muted, EXECUTING = blue, AWAITING_APPROVAL = yellow, COMPLETED = green, FAILED = red, HALTED = orange, STUCK = pale red).
  - Header row: title (first 80 chars), status pill, step count, elapsed time.
  - Click to expand:
    - **Creative ledger**: facts (yellow list), missing (muted list), plan (numbered list), currentDispatch (executor pill + task line + proposed spend if PAYMENT).
    - **Payment ledger**: vertical timeline of `PlacementEntry` rows. Each row shows placement ID, task, approval status pill (`PENDING` yellow, `APPROVED` green, `REJECTED` red), approved-by (when applicable), and — once the payment has executed — a `PaymentResult` block with the tx hash (linked) or failure reason.
    - **Creative entries**: timeline of `CreativeEntry` rows with executor pill, task, verdict pill (`OK` green, `BLOCKED_BY_GUARDRAIL` yellow, `FAILED` red), and collapsed `<pre>` of `sanitizedContent` (click to expand). Redacted spans render in italics with a tooltip showing the redaction tag (e.g., `[REDACTED:wallet-private-key]`).
    - **Report** (only when status is COMPLETED): summary paragraph + placements list + total spent in ETH.
    - **Failure or halt reason** (when applicable): one-paragraph block coloured by status.
