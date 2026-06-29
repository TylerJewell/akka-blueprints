# UI mockup — code-search-rag-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: CodeSearch Demo Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Code<span class="accent">Search</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Select a corpus tag (akka-http / akka-streams / akka-persistence) or leave at "All".
  3. Type a question and click **Search**.
  4. Watch the card transition through CHUNKS_SANITIZED → ANSWERING → ANSWERED → GROUNDED.
- Card **How it works**: one paragraph on submit → sanitize-chunks → answer → grounding; one paragraph on the secret sanitizer governance mechanism.
- Card **Components**: rows per component (QueryEntity, ChunkSanitizer, QueryWorkflow, CodeSearchAgent, GroundingScorer, ChunkIndexView, QueryEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Scorer as a supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — rendered with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `source-code: true` declaration in Data is filled and prominent. `decisions.authority_level = informational-only` and `oversight.human_in_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with one row (S1). ID badge coloured green (sanitizer).
- The row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Ask a question. <span class="accent">Find the code.</span>`. Subtitle: `One agent, secrets sanitized before it sees a line.`
- Layout: two-column.
  - **Left column** — Query submission panel + live list.
    - Submission panel: dropdown `Corpus tag` (All / akka-http / akka-streams / akka-persistence), `Question` textarea (with placeholder "e.g. Where is the connection pool size configured?"), `Submitted by` text input, and a yellow `Search` button.
    - Live list below: one card per query, newest-first. Each card shows status pill, grounding score chip (when grounding landed), question text truncated to 60 chars, age.
  - **Right column** — Selected-query detail.
    - Header: status pill + grounding score chip + question text.
    - Sanitized chunks list: a compact table with `chunkId`, `filePath`, `startLine`–`endLine`, `language`, and a small badge per secret category found (if any). If no secrets were found, the badge row is omitted.
    - Answer paragraph: the agent's 1–4-sentence prose answer.
    - References table: columns file path, start line, end line, language, relevance blurb.
    - Grounding section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red with a tooltip "Cited paths were not found in retrieved chunks."
- Status pill colours: SUBMITTED=muted, CHUNKS_SANITIZED=blue, ANSWERING=yellow, ANSWERED=blue, GROUNDED=green, FAILED=red.
- Grounding score chip: 5=green, 4=teal, 3=yellow, 2=orange, 1=red.

Raw chunk content is never displayed on this screen — only chunk metadata (file path, line range, language). A developer who needs to inspect raw content fetches `/api/queries/{id}` and reads `request.retrievedChunks[*].content` from the JSON. This is intentional: the UI demonstrates that the model received sanitized content, while the audit trail preserves the raw form.
