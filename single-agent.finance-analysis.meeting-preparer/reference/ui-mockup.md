# UI mockup — meeting-preparer

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Meeting Preparer</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Meeting<span class="accent">Preparer</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded counterparty (or fill in your own) and click **Prepare brief**.
  3. Watch the card transition through DATA_SANITIZED → BRIEFING → BRIEF_READY → EVALUATED.
  4. Click the card to expand the full brief — talking points, risk flags, financial highlights, and recent news.
- Card **How it works**: one paragraph on request → sanitize CRM → brief generation → eval; one paragraph on the governance mechanism (PII sanitizer).
- Card **Components**: rows per component (BriefingEntity, ContactSanitizer, BriefingWorkflow, BriefingAgent, BriefingEvaluator, BriefingView, BriefingEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Evaluator as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. Deployer fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (S1). ID badge coloured green (sanitizer).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Prepare for your call. <span class="accent">Read the brief.</span>`. Subtitle: `One agent, PII sanitized before it runs.`
- Layout: two-column.
  - **Left column** — Request panel + live list.
    - Request panel: text input `Counterparty name`, datetime input `Meeting date/time`, text input `Agenda topics (comma-separated)`, text input `Requested by`, and a yellow `Prepare brief` button. A "Load seeded counterparty" dropdown auto-fills all fields from the three seeded profiles.
    - Live list below: one card per brief, newest-first. Each card shows status pill, eval score chip (when eval landed), counterparty name, and age.
  - **Right column** — Selected-brief detail.
    - Header: status pill + eval score chip + `counterpartyName` + meeting date/time.
    - Meeting metadata: attendees list, agenda topics.
    - Sanitized CRM preview: a monospace block of the redacted CRM snapshot with PII category chips above (`person-name`, `email`, `phone`, etc.).
    - Executive summary paragraph.
    - Talking points: numbered list with topic label, point text, and an italic evidence-source tag.
    - Risk flags: amber/red chips with label, level badge, and one-sentence detail.
    - Financial highlights: a single line in a muted block.
    - Recent news: a bulleted list of headline strings.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: REQUESTED=muted, DATA_SANITIZED=blue, BRIEFING=yellow, BRIEF_READY=blue, EVALUATED=green, FAILED=red.
- Risk level chip colours: LOW=muted, MEDIUM=yellow, HIGH=red.
- Eval score chip colours: 4–5=green, 3=yellow, 1–2=red.

The raw CRM snapshot (including unredacted contact name, email, and phone) is never displayed on this screen — only the sanitized form. Requesters who need the raw values fetch `/api/briefs/{id}` and read `request.crmSnapshot` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
