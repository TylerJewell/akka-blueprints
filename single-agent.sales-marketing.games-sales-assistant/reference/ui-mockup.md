# UI mockup — games-sales-assistant

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Video Games Sales Assistant</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Video Games <span class="accent">Sales Assistant</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Click **New session**, enter a shopper id.
  3. Type a question (e.g., "What action RPGs do you have under $40 on PC?") and click **Ask**.
  4. Watch the turn transition through PENDING → ANSWERED with recommendation cards.
- Card **How it works**: one paragraph on session creation → turn posting → agent call → guardrail → response recorded; one paragraph on the one governance mechanism (before-agent-response catalog-and-content guardrail).
- Card **Components**: rows per component (SessionEntity, CatalogConsumer, CatalogIndex, SessionWorkflow, SalesAssistantAgent, ResponseGuardrail, SessionView, SessionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 CatalogIndex as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `decisions.authority_level = recommend-only` and `guardrail_blocks_forbidden_responses: true` declarations are the distinctive answers. `children-data`, `jurisdictions_in_scope`, and other deployer-specific fields show in muted italic with `TO_BE_COMPLETED_BY_DEPLOYER`.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (G1). ID badge coloured red (guardrail).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Get a recommendation.</span>`. Subtitle: `One agent, one guardrail, zero off-catalog surprises.`
- Layout: two-column.
  - **Left column** — Session panel + live list.
    - Session panel: `Shopper ID` text input and a yellow **New session** button. Below that, the live list of sessions, newest-first. Each session card shows status pill, shopper id, turn count chip, age.
    - Clicking a session card loads it in the right column.
  - **Right column** — Conversation view.
    - Header: session id + status pill + shopper id.
    - Question input: `Ask` textarea + yellow **Ask** button (disabled when no session selected or a turn is PENDING).
    - Turn list, newest-first. Each turn:
      - Shopper question in a left-aligned bubble.
      - Agent answer paragraph in a right-aligned bubble with status chip (PENDING spinner / ANSWERED / GUARDRAIL_REJECTED / FAILED).
      - Recommendation cards (when present): a horizontal scroll row of cards, each showing the game title, platform chip, price, and the one-sentence pitch.
      - Order history block (when present): a compact table with order id, title name, price paid, and purchase date.
- Status pill colours: CREATED=muted, ACTIVE=green, CLOSED=blue, FAILED=red.
- Turn status chip colours: PENDING=yellow (pulsing), ANSWERED=green, GUARDRAIL_REJECTED=orange, FAILED=red.
- Recommendation card: dark background, title in white, platform chip in blue, price in yellow, pitch in muted text.

The catalog data visible to the shopper (title names, prices) comes from the agent's response, not from a separate catalog API call. This keeps the UI stateless with respect to catalog state — the agent and guardrail are the authoritative source.
