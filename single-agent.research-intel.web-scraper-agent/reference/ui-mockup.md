# UI mockup — web-scraper-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: WebScraperAgent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Web<span class="accent">ScraperAgent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded URL (news-article / product-doc / tech-blog) and load it with one click, or type your own.
  3. Click **Submit scrape**.
  4. Watch the card transition through FETCHING → EXTRACTED → SANITIZED.
- Card **How it works**: one paragraph on submit → guardrail → fetch → extract → sanitize; one paragraph on the two governance mechanisms (before-tool-call guardrail, PII sanitizer).
- Card **Components**: rows per component (ScrapeEntity, ContentSanitizer, ScrapeWorkflow, WebScraperAgent, UrlGuardrail, FetchPageTool, ScrapeView, ScrapeEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 Tool as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `capabilities.untargeted-image-scraping = false` and `operations.rate_limit_per_origin_per_60s = 5` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a URL. <span class="accent">Read the extraction.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Seeded URL` (news-article / product-doc / tech-blog fixtures), `Target URL` text input (pre-fills from dropdown), `Extraction schema` dropdown (news-article / product-doc / tech-blog / custom), `Submitted by` text input, and a yellow `Submit scrape` button.
    - Live list below: one card per scrape, newest-first. Each card shows status pill, target URL (truncated to 40 chars), and age.
  - **Right column** — Selected-scrape detail.
    - Header: status pill + target URL + extraction schema.
    - If BLOCKED: a prominent red banner with the blocked reason.
    - Sanitized summary: the agent's 2-4-sentence paragraph, PII redaction tokens highlighted.
    - PII category chips above the summary (`email`, `phone`, `person-name`, etc.).
    - Data-points table: columns field name, value, source quote (italic). PII tokens in cells shown in a muted red colour.
    - Raw result link at the bottom: "View raw extraction via GET /api/scrapes/{id}" — opens the JSON in a new tab.
- Status pill colours: SUBMITTED=muted, FETCHING=yellow, EXTRACTED=blue, SANITIZED=green, BLOCKED=red, FAILED=red.
- The sanitized form is what the UI renders by default. The raw `result` is accessible only via the API link — this is intentional, demonstrating that downstream consumers receive only the redacted form.
