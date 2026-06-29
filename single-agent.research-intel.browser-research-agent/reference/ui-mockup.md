# UI mockup — reddit-search

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: RedditSearch</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Reddit<span class="accent">Search</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded topic or type your own. Set subreddit scope (optional) and page budget.
  3. Click **Start research**.
  4. Watch the card progress through BROWSING → REPORT_READY (or BUDGET_EXHAUSTED for a tight budget).
- Card **How it works**: one paragraph on submit → browse → score → report; one paragraph on the two governance mechanisms (navigation guardrail, budget halt).
- Card **Components**: rows per component (ResearchJobEntity, ResearchWorkflow, BrowserResearchAgent, NavigationGuardrail, BudgetEnforcer, RelevanceScorer, ResearchView, ResearchEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 1 Guardrail + 2 supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. `browser_accesses_third_party_content: true` and `external_tool_calls: [llm-completion, headless-browser-navigation, headless-browser-page-read]` are the distinctive entries. Many fields are `TO_BE_COMPLETED_BY_DEPLOYER` and displayed in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 orange (halt).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a topic. <span class="accent">Read the report.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live job list.
    - Submission panel: `Topic` text input (with a seeded-topic dropdown offering 3 presets), `Subreddit scope` text input (comma-separated, placeholder `(all subreddits)`), `Max pages` number input (1–20, default 10), and a green `Start research` button.
    - Live list below: one card per job, newest-first. Each card shows status pill, page-visit counter, topic truncated to 60 chars, age.
  - **Right column** — Selected-job detail.
    - Header: status pill + page-visit counter `N / maxPages pages` + topic.
    - Topic config recap: subreddit scope chips, max-pages badge.
    - Post list: ranked table with columns rank, title (link), subreddit chip, upvotes, summary sentence. When `posts` is empty, show `noResultsReason` instead.
    - Theme tags: a row of phrase chips, each showing the phrase and occurrence count.
    - Sentiment bar: three segments (positive / neutral / negative) with percentage labels.
    - Budget note (only for BUDGET_EXHAUSTED): amber banner "Page budget exhausted — showing partial results (N pages visited)."
- Status pill colours: QUEUED=muted, BROWSING=yellow, REPORT_READY=green, BUDGET_EXHAUSTED=amber, FAILED=red.
- Subreddit chip colour: blue.
- Post rank number: accent colour.
