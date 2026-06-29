# UI mockup — currency-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: CurrencyAgent</title>`.

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
- Headline: `Currency<span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Enter a currency pair (e.g. `USD` → `EUR`) and an amount, or load a seeded example.
  3. Click **Convert**.
  4. Watch the card transition through RATE_ATTACHED → CONVERTING → RESULT_RECORDED → EVALUATED.
- Card **How it works**: one paragraph on request → rate-fetch → agent conversion → freshness eval; one paragraph on the baseline wiring (no controls, ResultGuardrail for structural safety, FreshnessScorer for data-quality signal).
- Card **Components**: rows per component (ConversionEntity, RateFetcher, ConversionWorkflow, ExchangeRateAgent, ResultGuardrail, FreshnessScorer, ConversionView, ConversionEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: false` and `decisions.authority_level = information-only` declarations are the distinctive answers for this domain. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Table header with columns: ID, Name, Mechanism, Enforcement, Regulation Anchors.
- Body: one row reading "No controls defined in this baseline tier. Edit `eval-matrix.yaml` to add controls." in a muted full-width cell.
- A note below the table: "This is the baseline cell. Add entries to `eval-matrix.yaml` to populate this table."

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Enter a pair. <span class="accent">Get the rate.</span>`. Subtitle: `One agent, rate context as an attachment.`
- Layout: two-column.
  - **Left column** — Conversion form + live list.
    - Conversion form: `From` currency text input (placeholder `USD`), `To` currency text input (placeholder `EUR`), `Amount` numeric input, `Snapshot` dropdown (`spot` / `morning-fix` / `end-of-day`), `Requested by` text input, and a yellow `Convert` button. A "Load seeded example" link fills all fields with a USD→EUR spot example.
    - Live list below: one card per conversion, newest-first. Each card shows status pill, converted-amount chip (when result landed), freshness eval score chip (when eval landed), currency pair label, age.
  - **Right column** — Selected-conversion detail.
    - Header: status pill + currency pair + amount.
    - Rate snapshot block: label, rate, capturedAt timestamp, source tag.
    - Result block: large converted-amount display, rate-applied line, confidence note (italic), market-context paragraph.
    - Freshness eval section: a 1–5 score chip (colour by band: 1–2 = amber, 3 = blue, 4–5 = green) and the one-line rationale. Score ≤ 2 highlights the card border amber.
- Status pill colours: REQUESTED=muted, RATE_ATTACHED=blue, CONVERTING=yellow, RESULT_RECORDED=blue, EVALUATED=green, FAILED=red.
- Freshness score chip colours: 1–2=amber, 3=blue, 4–5=green.
