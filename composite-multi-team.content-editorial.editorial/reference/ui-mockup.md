# UI mockup — editorial-desk

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Editorial Desk</title>`.

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
- Headline: `Editorial <span class="accent">Desk</span>`. **No subtitle.**
- Card **Try it**: a `/akka:build` (Claude Code) block, then three numbered steps (submit a story, watch the desks run, watch the article publish). No env-var export block — `/akka:specify` handled the key during generation.
- Card **How it works**: one paragraph naming the editor-in-chief pipeline and the three desks' different coordination capabilities (research delegates, writing is a team over a shared board, review is a moderated panel).
- Card **Components**: table with rows for each component listed in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, event-sourced = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, document state machine, entity model — then the per-component detail below.`
- Stat tiles: count of each component kind (6 agents, 2 workflows, 3 event-sourced entities, 2 views).
- Four mermaid cards (component graph, sequence, document state machine, ER) with the Akka theme variables AND the Lesson 24 state-label CSS overrides (white state names, edge-label `foreignObject overflow:visible`, `transitionLabelColor #cccccc`). Without these the state names render black-on-black and the arrow labels clip.
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
- ID badges carry a colored mechanism pill: guardrail = red, eval-event = blue, hotl = muted. The four controls are G1 (guardrail · before-agent-response), G2 (guardrail · before-tool-call), E1 (eval-event · on-decision-eval), HO1 (hotl · live-compliance-review).
- Rows expand vertically on click to show rationale + implementation; one open at a time.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a topic. <span class="accent">Watch the desk run it.</span>` Subtitle: `The simulator also drips a topic every 60 s so the board is never empty.`
- Form card: a "Story topic" field, an optional "Requested by" field, a `Submit` button (yellow).
- The story board: one column or card lane per `StoryStatus` (Submitted · Assigned · Researching · Researched · Writing · Drafted · Reviewing · Approved · Published), or a single list with a status chip per card. Each story card shows:
  - the topic and status chip (status colours: submitted muted, writing/reviewing yellow, approved green, published blue);
  - the brief angle once assigned;
  - the research digest summary once researched;
  - a section-progress line (e.g., "3 / 4 sections written") during writing, drawn from the section board;
  - the review verdict chip (PASS green / REVISE red) and any `mustRevise` titles;
  - the three stage-eval scores (research / draft / review) as small score chips;
  - once published: the article headline, the published URL, and a **compliance-review box** — a small form (reviewed-by, an outcome toggle CLEARED/FLAGGED, a comments field, a Submit) that posts to `POST /api/stories/{id}/compliance-review`; once a review exists, the box shows the recorded outcome and comments instead.
- The section-board panel: grouped by `SectionStatus` (Open · Claimed · Written). Each section card shows its title, the story it belongs to, the claiming writer (when claimed), and the word count (when written). Cards move between groups live via SSE.
- Both the story board and the section board update live — the story board over `/api/stories/sse`, the section board over `/api/sections/sse` — with no page reload.
