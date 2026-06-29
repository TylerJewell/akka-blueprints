# UI mockup

## Tab switching — MUST be attribute-based
Five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. Five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching attributes — never by NodeList index. Canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any removed panel must be deleted; `display:none` is not enough. See Lesson 26.

## Tab 1 — Overview
- Eyebrow: Overview
- Headline: Blackboard <span class="accent">Knowledge Discovery</span>. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, three numbered steps (submit an inquiry, watch the control shell call specialists, watch the blackboard converge). No env-var export block — `/akka:specify` handled the key during generation.
- Card **How it works**: one paragraph naming components and the read-write-via-entity coordination primitive.
- Card **Components**: table with rows for each component in SPEC.md §4. Kind column coloured per component palette (agent = blue, workflow = purple, event-sourced = yellow, key-value = gold, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture
- Eyebrow: Architecture. Headline: What gets <span class="accent">wired together</span>. Subtitle: Component graph, sequence, blackboard state machine, entity model — per-component detail below.
- Stat tiles: count of each component kind.
- Four mermaid cards (component graph, sequence, blackboard state machine, ER) with Akka theme variables AND Lesson 24 state-label CSS overrides (white state names, `g.edgeLabel foreignObject { overflow: visible; }`, `transitionLabelColor: #cccccc`). Without these state names render dark-on-dark and edge labels clip.
- Compressed comp-row table: one row per component, click expands to show description plus a short Java source snippet (10-20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey
- Eyebrow: Risk Survey. Headline: What this <span class="accent">deployer would declare</span>.
- Subtitle: Seven sections mirror the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.
- 7 sections in order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each question block rendered with its question text and chips/textareas/list widgets in their selected state per `risk-survey.yaml`.
- Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer"); unanswered blocks get `opacity: 0.45`.

## Tab 4 — Eval Matrix
- Eyebrow: Eval Matrix. Headline: Controls the <span class="accent">runtime enforces</span>. Subtitle: Each row is one governance control. Click for rationale and implementation.
- 5-column table: ID | Control (obligation) | Mechanism | Implementation | Source.
- ID badges carry a colored mechanism pill: guardrail = red, eval-event = teal.
- Rows expand vertically on click to show rationale + implementation; one open at a time.

## Tab 5 — App UI
- Eyebrow: App UI. Headline: Ask a question. <span class="accent">Watch the board fill in.</span> Subtitle: The simulator also drips an inquiry every 90 s so the board is never empty.
- Form card: "Research question" textarea, Submit button (yellow).
- Control strip: Halt / Resume toggle showing the current `SystemControl` state; when halted, a red banner reads "Shell halted by {by}: {reason}".
- Per-inquiry panel (one card per active blackboard): question, status, iteration count; sub-panels for Sources, Claims, Hypotheses, Disputed, Open sub-questions, Report. Each contribution chip shows its score from `ContributionScorer`. A rejected-writes strip lists any recent `WriteRejected` events with reason.
- Panels reorder via SSE so the most recently active inquiry stays at the top.
