# UI mockup — memory-bank

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: MemoryBank</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Memory<span class="accent">Bank</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Select a namespace (personal-assistant, project-notes, or custom) and type a fact to remember.
  3. Click **Store** and watch the card transition through SANITIZED → STORING → STORED.
  4. Type a recall query and click **Recall** to see ranked matching entries.
- Card **How it works**: one paragraph on submit → sanitize → agent store/recall → SSE update; one paragraph on the sanitizer governance mechanism and why raw content never reaches the model.
- Card **Components**: rows per component (MemoryEntity, MemorySanitizer, MemoryWorkflow, MemoryAgent, MemoryView, MemoryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = autonomous` and `oversight.human_in_loop = false` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (S1). ID badge coloured green (sanitizer).
- Row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Store a fact. <span class="accent">Recall it later.</span>`. Subtitle: `One agent. PII stripped before the model ever sees the content.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: operation toggle (REMEMBER / RECALL), namespace dropdown (personal-assistant / project-notes / custom), `Content` textarea (with a "Load seeded example" link), `Submitted by` text input, and a yellow `Store` or `Recall` button (label changes with the toggle).
    - Live list below: one card per memory entry, newest-first. Each card shows status pill, operation badge (REMEMBER/RECALL), namespace chip, content preview (first 60 chars), age.
  - **Right column** — Selected-entry detail.
    - Header: status pill + operation badge + namespace chip + content preview.
    - Sanitized content block: a monospace display of the redacted content, with PII category chips above (e.g., `email`, `phone`) or a muted "no PII found" label.
    - For REMEMBER entries: extracted tags as chips, acknowledgement text from the agent.
    - For RECALL entries: ranked match list — each match shows relevance score bar, stored content snippet, tags, timestamp.
    - Raw content is not shown. The reviewer can fetch `/api/memories/{id}` and read `request.rawContent` from the JSON if the audit trail is needed.
- Status pill colours: SUBMITTED=muted, SANITIZED=blue, STORING=yellow, STORED=green, FAILED=red.
- Operation badge colours: REMEMBER=purple, RECALL=teal.
- Namespace chip colours: PERSONAL_ASSISTANT=indigo, PROJECT_NOTES=amber, CUSTOM=slate.
