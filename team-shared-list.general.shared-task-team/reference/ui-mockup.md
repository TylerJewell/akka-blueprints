# UI mockup — shared-task-team

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Shared Task Team</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents (clicking a tab and seeing a blank because a hidden zombie panel occupied the index).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Shared <span class="accent">Task Team</span>`. **No subtitle.**
- Card **Try it**: a `/akka:build` (Claude Code) block, then three numbered steps (submit a goal, watch the planner fill the board, watch workers claim and complete tasks). No env-var export block — `/akka:specify` handled the key during generation.
- Card **How it works**: one paragraph naming the components and the atomic-claim coordination.
- Card **Components**: table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, event-sourced = yellow, key-value = gold, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, task state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind.
- Four mermaid cards (component graph, sequence, task state machine, ER) with the Akka theme variables AND the Lesson 24 state-label CSS overrides (white state names, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`). Without these the state names render black-on-black and the arrow labels clip.
- Compressed comp-row table: one row per component, click expands to show the description plus a short Java source snippet (10–20 lines) with hand-tagged `<kw>`/`<ty>`/`<st>`/`<cm>`/`<an>`/`<fn>`/`<num>` syntax-highlight spans.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Subtitle: `Seven sections mirroring the canonical questionnaire. Sample-known answers filled in; deployer-specific fields faded.`
- 7 sections in this order: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each question block is rendered with the question text and the chips / textareas / list widgets in their selected state per `risk-survey.yaml`.
- Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer"); unanswered blocks get `opacity: 0.45`.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges carry a colored mechanism pill: guardrail = red, eval-event = pale blue, halt = red.
- Rows expand vertically on click to show rationale + implementation; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a goal. <span class="accent">Watch the team complete it.</span>` Subtitle: `The simulator also drips a goal every 60 s so the board is never empty.`
- Form card: a "Goal title" field, a "Description" textarea, a `Submit` button (yellow).
- Control strip: a Halt / Resume toggle showing the current `SystemControl` state; when halted, a red banner reads "Team halted by {by}: {reason}".
- The board: one column per `TaskStatus` (Open · Claimed · In progress · In review · Done · Blocked). Each task is a card showing its title, the goal it belongs to, the claiming worker (when claimed), the artifact summary (when present), and the quality result chip (green pass / red fail). A `BLOCKED` card shows the block reason. Cards move between columns live via SSE.
- Peer mailbox panel: a per-worker list of `PeerMessage` items; an unanswered message shows a reply box. Posting a reply unblocks the waiting task.
