# UI mockup — gitty

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Gitty</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Git<span class="accent">ty</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Paste a GitHub issue or PR URL, or pick a seeded example.
  3. Add an optional context note, then click **Queue draft**.
  4. Watch the card transition through THREAD_FETCHED → DRAFTING → PENDING_REVIEW, then Approve or Discard.
- Card **How it works**: one paragraph on queue → fetch → draft → HITL; one paragraph on the two governance mechanisms (tone guardrail, maintainer approval gate).
- Card **Components**: rows per component (DraftEntity, ThreadFetcher, DraftWorkflow, ReplyDrafterAgent, ToneGuardrail, DraftView, DraftEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive filled answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 amber (HITL).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Queue a thread. <span class="accent">Read the draft.</span>`. Subtitle: `One agent, one guardrail, one human gate.`
- Layout: two-column.
  - **Left column** — Queue panel + live list.
    - Queue panel: `Thread URL` text input (placeholder: `https://github.com/owner/repo/issues/N`), `Context note` textarea (optional, placeholder: "e.g. We fixed this in v3 — be brief"), `Queued by` text input, and a yellow `Queue draft` button. A "Load seeded example" link fills the URL and context note from one of three seeded threads.
    - Live list below: one card per draft, newest-first. Each card shows status pill, tone badge (when draft landed), draft title (thread title), age.
  - **Right column** — Selected-draft detail.
    - Header: status pill + tone badge + `threadTitle` + link icon to the original GitHub URL.
    - Thread info: repo slug, kind chip (ISSUE / PR), comment count, fetched timestamp.
    - Context note: the maintainer's original hint, if any.
    - Draft text: the agent's reply, displayed in an editable textarea (editable only in `PENDING_REVIEW` state). Character count badge (turns red above 800).
    - References list: chips showing what the draft cited.
    - Action buttons (visible in `PENDING_REVIEW` only): green **Approve** button, red **Discard** button.
    - Approved state: readonly textarea with approved text + a **Copy to clipboard** button.
    - Discarded state: a muted "Discarded by {actorId}" note.
- Status pill colours: QUEUED=muted, THREAD_FETCHED=blue, DRAFTING=yellow, PENDING_REVIEW=amber, APPROVED=green, DISCARDED=muted, FAILED=red.
- Tone badge colours: helpful=blue, closing=green, asking-for-info=yellow, acknowledging=muted.
