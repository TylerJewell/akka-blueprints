# SPEC — web-scraper-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** WebScraperAgent.
**One-line pitch:** A user submits a URL and an optional extraction schema; one AI agent fetches the page via an HTTP tool call and returns a structured `ScrapeResult` — title, summary, and a list of key data points extracted from the page content.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `WebScraperAgent` (AutonomousAgent) carries the entire fetch-and-extract decision; the surrounding components only gate its input and clean its output. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs before every invocation of the `fetchPage` tool inside the agent's task. It validates the target URL's domain against an allowlist and checks that the origin has not exceeded its per-window scrape budget. A blocked URL causes the guardrail to return a structured rejection; the agent receives the rejection as tool output and surfaces an explanatory `ScrapeResult` with `status = BLOCKED` — the fetch never happens.
- A **PII sanitizer** runs inside a Consumer between the raw content extraction and the stored result — so downstream consumers and the UI never see personal identifiers that happened to be on the scraped page.

The blueprint shows that a tool-calling agent can carry meaningful governance around each tool invocation, not just around the final output.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a URL into the **Target URL** input (or picks one of three seeded examples — a public news article fixture, a product-documentation page fixture, a tech-blog fixture).
2. The user selects an **extraction schema** from a dropdown (news-article, product-doc, tech-blog) or enters a free-text extraction goal.
3. The user clicks **Submit scrape**. The UI POSTs to `/api/scrapes` and receives a `scrapeId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `FETCHING` if the URL passes the guardrail, or `BLOCKED` immediately if the domain is not in the allowlist or the rate limit is exhausted.
5. Within ~10–30 s (live LLM) or ~2 s (mock), the agent completes extraction. The card transitions to `EXTRACTED`. The result appears: a `title`, a `summary` paragraph, and a `dataPoints` table (field name, value, source quote from the page).
6. Within ~1 s of extraction, the sanitizer Consumer fires. The card transitions to `SANITIZED`. The stored result shows `[REDACTED-*]` tokens in place of any personal data the sanitizer found, together with a `piiCategoriesFound` chip list.
7. The user can submit another URL; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ScrapeEndpoint` | `HttpEndpoint` | `/api/scrapes/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ScrapeEntity`, `ScrapeView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ScrapeEntity` | `EventSourcedEntity` | Per-scrape lifecycle: submitted → fetching → extracted → sanitized (or blocked/failed). Source of truth. | `ScrapeEndpoint`, `ContentSanitizer`, `ScrapeWorkflow` | `ScrapeView` |
| `ContentSanitizer` | `Consumer` | Subscribes to `RawContentExtracted` events; redacts PII; calls `ScrapeEntity.attachSanitized`. | `ScrapeEntity` events | `ScrapeEntity` |
| `ScrapeWorkflow` | `Workflow` | One workflow per scrape. Steps: `submitStep` → `scrapeStep` → `awaitSanitizedStep`. | started by `ScrapeEndpoint` on submit | `WebScraperAgent`, `ScrapeEntity` |
| `WebScraperAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the URL and extraction schema as task instructions; calls the `fetchPage` tool (guarded by `UrlGuardrail`); returns `ScrapeResult`. | invoked by `ScrapeWorkflow` | returns result |
| `UrlGuardrail` | (guardrail class) | `before-tool-call` hook on `WebScraperAgent.fetchPage`. Validates domain allowlist and per-origin rate-limit window. | registered on agent | blocks or passes tool invocation |
| `ScrapeView` | `View` | Read model: one row per scrape for the UI. | `ScrapeEntity` events | `ScrapeEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ScrapeRequest(
    String scrapeId,
    String targetUrl,
    String extractionSchema,   // "news-article" | "product-doc" | "tech-blog" | free-text
    String submittedBy,
    Instant submittedAt
) {}

record DataPoint(
    String fieldName,
    String value,
    String sourceQuote   // verbatim passage from the page that supports the value
) {}

record ScrapeResult(
    String title,
    String summary,
    List<DataPoint> dataPoints,
    String fetchedUrl,         // resolved URL after any redirect
    int httpStatus,
    Instant extractedAt
) {}

record SanitizedResult(
    String sanitizedSummary,
    List<DataPoint> sanitizedDataPoints,   // values and quotes with PII tokens replaced
    List<String> piiCategoriesFound
) {}

record Scrape(
    String scrapeId,
    Optional<ScrapeRequest> request,
    Optional<ScrapeResult> result,
    Optional<SanitizedResult> sanitized,
    ScrapeStatus status,
    Optional<String> blockedReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ScrapeStatus {
    SUBMITTED, FETCHING, EXTRACTED, SANITIZED, BLOCKED, FAILED
}
```

Events on `ScrapeEntity`: `ScrapeSubmitted`, `ScrapeFetching`, `RawContentExtracted`, `ContentSanitized`, `ScrapeBlocked`, `ScrapeFailed`.

Every nullable lifecycle field on the `Scrape` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/scrapes` — body `{ targetUrl, extractionSchema, submittedBy }` → `{ scrapeId }`.
- `GET /api/scrapes` — list all scrapes, newest-first.
- `GET /api/scrapes/{id}` — one scrape.
- `GET /api/scrapes/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: WebScraperAgent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted scrapes (status pill + age + target URL truncated) and a right pane with the selected scrape's detail — target URL, extraction schema, raw and sanitized summary, data-points table, PII category chips.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs before every `fetchPage` tool invocation inside `WebScraperAgent`. The `UrlGuardrail` class is registered on the agent's guardrail-configuration block, bound to the `before-tool-call` hook. It (1) parses the target URL, (2) checks the domain against the seeded allowlist (loaded from `src/main/resources/sample-events/allowed-domains.jsonl`), (3) checks that the origin has not exceeded its rolling 60-second scrape budget (default: 5 requests per origin per window). On any failure it returns a structured rejection with a `reason` field; the agent receives the rejection as tool output, sets `ScrapeResult.status = BLOCKED`, and the workflow writes a `ScrapeBlocked` event. The fetch HTTP call never fires.
- **S1 — PII sanitizer** (`pii`, applied inside `ContentSanitizer` Consumer): runs after `RawContentExtracted`. Redacts emails, phone numbers, person names, postal addresses, and account-like identifiers from `ScrapeResult.summary` and from every `DataPoint.value` and `DataPoint.sourceQuote`. Records which categories were found in `piiCategoriesFound`. The raw `ScrapeResult` is preserved on the entity for audit; the sanitized form is what the UI displays.

## 9. Agent prompts

- `WebScraperAgent` → `prompts/web-scraper.md`. The single decision-making LLM. System prompt instructs it to call the `fetchPage` tool with the submitted URL, parse the returned HTML, and extract the requested fields into a `ScrapeResult`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits an allowed URL with the `news-article` schema; within 30 s the extraction appears with title, summary, and at least 3 data points each carrying a non-empty `sourceQuote`.
2. **J2** — User submits a URL whose domain is not in the allowlist; the `before-tool-call` guardrail fires; the card reaches `BLOCKED` without any `fetchPage` HTTP call appearing in the service log.
3. **J3** — The page at the submitted URL contains `reporter@example.com` and a phone number; after sanitization the stored `sanitizedSummary` shows only `[REDACTED-EMAIL]` and `[REDACTED-PHONE]`; the entity's `result.summary` retains the raw text.
4. **J4** — User submits the same allowed domain six times within 60 s; the fifth and sixth submissions are blocked by the rate-limit branch of the guardrail.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named web-scraper-agent demonstrating the single-agent × research-intel cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-web-scraper-agent. Java package
io.akka.samples.aiagentthatcanscrapewebpages. Akka 3.6.0. HTTP port 9519.

Components to wire (exactly):

- 1 AutonomousAgent WebScraperAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/web-scraper.md>) and
  .capability(TaskAcceptance.of(SCRAPE_URL).maxIterationsPerTask(3))
  .tool(FetchPageTool.class). The task receives the target URL and extraction schema in
  its instruction text. The agent calls the fetchPage tool (one registered tool), which
  returns raw HTML as a String. Output: ScrapeResult{title: String, summary: String,
  dataPoints: List<DataPoint>, fetchedUrl: String, httpStatus: int, extractedAt: Instant}.
  The agent is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the
  tool call is replaced with a structured error that the agent surfaces in its final output.

- 1 tool class FetchPageTool — an Akka tool that accepts a URL String, performs an
  in-process HTTP GET (Java HttpClient, 10 s timeout), and returns the response body as
  a String (truncated to 50 000 chars). In mock mode the tool reads from
  src/main/resources/mock-responses/pages/<fixture-slug>.html instead of making a
  real HTTP call.

- 1 Workflow ScrapeWorkflow per scrapeId with three steps:
  * submitStep — calls ScrapeEntity.markFetching, then runs componentClient
    .forAutonomousAgent(WebScraperAgent.class, "scraper-" + scrapeId).runSingleTask(
      TaskDef.instructions(formatRequest(request))
    ) — returns a taskId, then forTask(taskId).result(SCRAPE_URL) to fetch the result.
    On BLOCKED status calls ScrapeEntity.block(reason). On success calls
    ScrapeEntity.recordResult(result). WorkflowSettings.stepTimeout 90s with
    defaultStepRecovery maxRetries(2).failoverTo(ScrapeWorkflow::error).
  * awaitSanitizedStep — polls ScrapeEntity.getScrape every 1 s; on
    scrape.sanitized().isPresent() advances to done. WorkflowSettings.stepTimeout 15s.
  * done step — no-op; the workflow terminates.
    error step transitions the entity to FAILED. WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ScrapeEntity (one per scrapeId). State Scrape{scrapeId: String,
  request: Optional<ScrapeRequest>, result: Optional<ScrapeResult>,
  sanitized: Optional<SanitizedResult>, status: ScrapeStatus,
  blockedReason: Optional<String>, createdAt: Instant, finishedAt: Optional<Instant>}.
  ScrapeStatus enum: SUBMITTED, FETCHING, EXTRACTED, SANITIZED, BLOCKED, FAILED.
  Events: ScrapeSubmitted{request}, ScrapeFetching{}, RawContentExtracted{result},
  ContentSanitized{sanitized}, ScrapeBlocked{reason}, ScrapeFailed{reason}.
  Commands: submit, markFetching, recordResult, attachSanitized, block, fail, getScrape.
  emptyState() returns Scrape.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 Consumer ContentSanitizer subscribed to ScrapeEntity events; on RawContentExtracted
  runs a regex+heuristic redaction pipeline over result.summary and each
  DataPoint.value and DataPoint.sourceQuote (emails, phone numbers, person-name
  heuristic, postal-address heuristic, account-id-like tokens), computes
  piiCategoriesFound, builds SanitizedResult, then calls ScrapeEntity.attachSanitized.

- 1 View ScrapeView with row type ScrapeRow (mirrors Scrape minus result.raw fields —
  the audit log keeps the raw; the view holds the sanitized form for the UI).
  Table updater consumes ScrapeEntity events. ONE query getAllScrapes: SELECT * AS scrapes
  FROM scrape_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ScrapeEndpoint at /api with POST /scrapes (body {targetUrl, extractionSchema,
    submittedBy}; mints scrapeId; calls ScrapeEntity.submit; starts ScrapeWorkflow;
    returns {scrapeId}), GET /scrapes (list from getAllScrapes, sorted newest-first),
    GET /scrapes/{id} (one row), GET /scrapes/sse (Server-Sent Events forwarded from
    the view's stream-updates), and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ScrapeTasks.java declaring one Task<R> constant: SCRAPE_URL = Task.name("Scrape URL")
  .description("Fetch the page at the given URL and extract structured data per the
  requested schema").resultConformsTo(ScrapeResult.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ScrapeRequest, DataPoint, ScrapeResult, SanitizedResult, Scrape,
  ScrapeStatus.

- UrlGuardrail.java implementing the before-tool-call hook. Reads the target URL from
  the pending tool invocation, runs the two checks listed in eval-matrix.yaml G1
  (domain allowlist, per-origin rate-limit window), and either passes the invocation
  through or returns Guardrail.reject(<structured-error>) to block the tool call and
  let the agent surface a BLOCKED result.

- AllowlistLoader.java — loads src/main/resources/sample-events/allowed-domains.jsonl
  at startup. Exposes isDomainAllowed(String domain) and checkRateLimit(String origin)
  methods used by UrlGuardrail.

- RateLimitStore.java — in-process ConcurrentHashMap keyed by origin, storing call
  timestamps in a rolling 60-second window. Thread-safe. Injected into UrlGuardrail.

- FetchPageTool.java — Akka tool registered on WebScraperAgent. In production mode
  uses Java HttpClient (10 s connect + read timeout). In mock mode reads from
  src/main/resources/mock-responses/pages/ by fixture slug.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9519 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/allowed-domains.jsonl with 10 seed entries:
  example.com, docs.example.org, news.example.net, techblog.example.io,
  products.example.co, api-docs.example.dev, blog.example.info,
  releases.example.app, wiki.example.site, status.example.page. These are fictional
  hostnames used by the bundled HTML fixtures.

- src/main/resources/mock-responses/pages/ containing three HTML fixture files:
  news-article.html (a 900-word synthetic news article with 2 PII strings — a reporter
  byline email and a contact phone number), product-doc.html (a 700-word product
  documentation page), tech-blog.html (a 600-word technical post with one person name
  and one email). Each is served by FetchPageTool in mock mode.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  capabilities.untargeted-image-scraping = false (the agent scrapes only explicitly
  submitted URLs), oversight.human_in_loop = false (extracted data is consumed
  programmatically by downstream tools; human review is optional), failure.failure_modes
  including "blocked-legitimate-domain", "pii-leakage-via-downstream",
  "rate-limit-false-positive", "extraction-hallucination"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/web-scraper.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Web Scraper Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of scrape cards; right = selected-scrape detail with target URL,
  extraction schema, sanitized summary, data-points table, PII category chips).
  Browser title exactly: <title>Akka Sample: WebScraperAgent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider block below).
        Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java. Per-task mock shapes for THIS blueprint:
    scrape-url.json — 6 ScrapeResult entries covering the three seeded fixture pages
    (news-article, product-doc, tech-blog), each with a title, a 2-3 sentence summary,
    and 4-6 DataPoints with non-empty sourceQuote fields. Plus 2 entries where
    dataPoints are empty (exercises the data-point completeness path). The mock selects
    entries deterministically by seedFor(scrapeId).
- FetchPageTool in mock mode reads from src/main/resources/mock-responses/pages/
  by fixture slug derived from the submitted URL's hostname; falls back to
  news-article.html if the slug is not found.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WebScraperAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ScrapeTasks.java MUST
  exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (submitStep 90s, awaitSanitizedStep 15s, done 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Scrape row record is Optional<T>.
- Lesson 7: ScrapeTasks.java with SCRAPE_URL = Task.name(...).description(...)
  .resultConformsTo(ScrapeResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9519 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WebScraperAgent).
  The PII sanitizer (ContentSanitizer) is a Consumer, not an agent — keeping the
  pattern's "one agent" promise honest.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, bound to the fetchPage tool invocation hook — not as an external check
  that runs before the workflow calls the agent.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
