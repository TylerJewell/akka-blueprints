# UI mockup — morning-email-debrief

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Morning Email Debrief</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Morning Email <span class="accent">Debrief</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the poller to deliver a batch, watch the pipeline progress, view the assembled digest).
- Card **How it works**: one paragraph on the poll → sanitize → summarize → assemble flow; one paragraph on the two governance mechanisms (PII sanitizer, periodic eval).
- Card **Components**: rows per component (poller, queue, sanitizer, summarizer, assembler, workflow, email entity, debrief entity, email view, debrief view, eval runner, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent, 1 AutonomousAgent, 1 Workflow, 3 ESEs, 2 Views, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, ER diagram).
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs matching governance.html with answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled; `decisions_surface: read-only-summary` and `human_in_loop: false` are the most distinctive. Many deployer-specific fields are faded as `TO_BE_COMPLETED_BY_DEPLOYER`.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (S1, E1). ID badges coloured: S1 cyan (sanitizer), E1 green (eval-periodic).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Read this morning's <span class="accent">email debrief.</span>`. Subtitle: `Emails arrive in batches every 60 s. The digest assembles automatically.`
- Layout: two-panel.
  - **Left panel** — Debrief run list, sorted newest-first. Each run card shows:
    - Header: status badge (CREATED / ASSEMBLING / READY / FAILED), run timestamp.
    - Progress indicator: "X of Y emails summarised" while assembling.
    - Eval score chip (when populated).
  - **Right panel** — selected debrief run detail.
    - `narrativeSummary` text block (read-only).
    - Per-entry table: one row per `DebriefEntry` with priority chip, category chip, one-liner summary.
    - Eval score + rationale section (shown when `evalScore` is populated).
    - "Email detail" expand: clicking any row opens the sanitized email subject and redacted body.
- Status badge colours: CREATED=muted, ASSEMBLING=yellow, READY=green, FAILED=red.
- Priority chip colours: HIGH=red, MEDIUM=yellow, LOW=muted.
- The raw `incoming` payload (pre-sanitization) is never displayed in the UI. The `EmailEntity.incoming` field is audit-only.
