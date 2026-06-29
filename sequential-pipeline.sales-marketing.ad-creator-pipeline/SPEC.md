# SPEC — ad-creator-pipeline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AI Ads Generator.
**One-line pitch:** A user submits a product page URL; one `AdCreatorAgent` walks it through three task phases — **SCRAPE** product data, **COPY** ad text for multiple formats, **VISUAL** an image-generation prompt — with each phase gated on the prior phase's recorded output and a brand-safety guardrail screening every outbound response before it is accepted.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a sales-marketing domain. One `AdCreatorAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the SCRAPE task's typed output (`ProductProfile`) becomes the COPY task's instruction context; the COPY task's typed output (`AdCopy`) becomes the VISUAL task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-agent-response` guardrail** (`BrandSafetyGuardrail`) sits between the agent and every response it emits. It scans outbound ad copy for disallowed claims (unsubstantiated superlatives, competitor brand names from a configurable list, prohibited product categories). A response containing a violation is blocked before it reaches the workflow; the agent receives a structured rejection and must rewrite within its iteration budget. This is the right cut for ad copy: the brand risk arrives in the agent's output, not in its tool calls.
- A **`before-tool-call` guardrail** (`ScrapingPolicyGuardrail`) sits in front of every browser-automation tool. It reads the target URL from the candidate tool call and checks it against the scraping allow-list recorded on `AdJobEntity` at job-creation time. A URL outside the allow-list is rejected before the tool body runs. The rejection returns a structured error to the agent loop; the agent is expected to fall back to the approved product URL. The same hook enforces phase order: only SCRAPE-phase tools are callable while the entity has not yet recorded `ProductScraped`.

The blueprint shows that a sequential pipeline often needs more than one governance hook — the `before-tool-call` hook guards the data-ingestion boundary, and the `before-agent-response` hook guards the content-output boundary. Both sit on the same agent and complement each other.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **product URL** into the input (or picks one of three seeded products — `Zenith Wireless Headphones`, `Alpine Trekking Boots`, `ClearFlow Water Bottle`).
2. The user clicks **Generate ad**. The UI POSTs to `/api/ad-jobs` and receives an `adJobId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `SCRAPING` — the workflow has started `scrapeStep` and the agent has been handed the SCRAPE task.
4. Within ~10–20 s the card reaches `COPYING` — the `ProductProfile` is visible in the card detail (product name, features list, target audience, brand constraints). The agent's SCRAPE task returned; the workflow recorded `ProductScraped` and ran the COPY task.
5. Within ~10–20 s more the card reaches `GENERATING_VISUAL`. The `AdCopy` is visible (headline, body, call-to-action, plus a format table showing the social-media variants). If the agent's first COPY attempt triggered the brand-safety guardrail, the rejection-log strip shows the blocked response.
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full `AdPackage` — all copy variants plus the image-generation prompt — and a quality score chip (1–5) with a one-line rationale.
7. The user can submit another product URL; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AdJobEndpoint` | `HttpEndpoint` | `/api/ad-jobs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `AdJobEntity`, `AdJobView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AdJobEntity` | `EventSourcedEntity` | Per-job lifecycle: created → scraping → scraped → copying → copied → generating-visual → visual-generated → evaluated. Source of truth. | `AdJobEndpoint`, `AdPipelineWorkflow` | `AdJobView` |
| `AdPipelineWorkflow` | `Workflow` | One workflow per ad job. Steps: `scrapeStep` → `copyStep` → `visualStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `AdJobEndpoint` after `CREATED` | `AdCreatorAgent`, `AdJobEntity` |
| `AdCreatorAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `AdTasks.java`: `SCRAPE_PRODUCT` → `ProductProfile`, `DRAFT_COPY` → `AdCopy`, `GENERATE_VISUAL` → `VisualSpec`. Each task is registered with the phase-appropriate function tools. | invoked by `AdPipelineWorkflow` | returns typed results |
| `ScrapeTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchProductPage(url)` and `extractAttributes(html)`. Reads from `src/main/resources/sample-data/products/*.json` for deterministic offline output. | called from SCRAPE task | returns `ProductProfile` |
| `CopyTools` | function-tools class | Implements `draftHeadline(profile)` and `draftBody(profile, format)`. In-memory template expansion. | called from COPY task | returns `AdCopy` |
| `VisualTools` | function-tools class | Implements `buildImagePrompt(profile, copy)` and `selectAspectRatio(format)`. | called from VISUAL task | returns `VisualSpec` |
| `ScrapingPolicyGuardrail` | `before-tool-call` guardrail (registered on `AdCreatorAgent`) | Reads the candidate tool call's target URL and the allow-list from the in-flight `AdJobEntity`. Rejects any browser tool whose URL is not on the allow-list. Also enforces phase order. | every tool call on every task | accept / structured-reject |
| `BrandSafetyGuardrail` | `before-agent-response` guardrail (registered on `AdCreatorAgent`) | Scans the agent's outbound response text for disallowed claims, competitor names, and prohibited categories. Blocks the response if a violation is detected; returns a structured rejection so the agent can rewrite. | every agent response | accept / structured-reject |
| `AdQualityScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `AdPackage`, `AdCopy`, `ProductProfile`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `AdJobView` | `View` | Read model: one row per ad job for the UI. | `AdJobEntity` events | `AdJobEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ProductAttribute(String name, String value) {}

record ProductProfile(
    String productName,
    String productUrl,
    List<ProductAttribute> attributes,
    String targetAudience,
    List<String> brandConstraints,
    Instant scrapedAt
) {}

record AdVariant(String format, String headline, String body, String callToAction) {}

record AdCopy(
    List<AdVariant> variants,
    String primaryHeadline,
    String primaryBody,
    String callToAction,
    Instant draftedAt
) {}

record VisualSpec(
    String imagePrompt,
    String aspectRatio,
    String styleGuidance,
    Instant generatedAt
) {}

record AdPackage(
    String title,
    AdCopy copy,
    VisualSpec visual,
    Instant assembledAt
) {}

record BrandViolation(String rule, String offendingText, String suggestion) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record AdJobRecord(
    String adJobId,
    Optional<String> productUrl,
    Optional<ProductProfile> profile,
    Optional<AdCopy> copy,
    Optional<VisualSpec> visual,
    Optional<AdPackage> adPackage,
    Optional<EvalResult> eval,
    AdJobStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AdJobStatus {
    CREATED, SCRAPING, SCRAPED, COPYING, COPIED,
    GENERATING_VISUAL, VISUAL_GENERATED, EVALUATED, FAILED
}
```

Events on `AdJobEntity`: `AdJobCreated`, `ScrapeStarted`, `ProductScraped`, `CopyStarted`, `CopyDrafted`, `VisualStarted`, `VisualGenerated`, `AdEvaluated`, `ScrapingRejected`, `BrandSafetyRejected`, `AdJobFailed`.

Every nullable lifecycle field on the `AdJobRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/ad-jobs` — body `{ productUrl }` → `{ adJobId }`.
- `GET /api/ad-jobs` — list all ad jobs, newest-first.
- `GET /api/ad-jobs/{id}` — one ad job.
- `GET /api/ad-jobs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: AI Ads Generator</title>`.

The App UI tab is a two-column layout: a left rail with the live list of ad jobs (status pill + product name + age) and a right pane with the selected job's detail — product URL, scraped attributes table, ad copy variants, visual spec, eval score chip, and a guardrail-rejection log strip if any scraping-policy or brand-safety rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (brand safety)**: `BrandSafetyGuardrail` is registered on `AdCreatorAgent` and runs before every agent response. It scans the response text for disallowed claims (superlatives without evidence: "best", "cheapest", "guaranteed"), competitor brand names (configurable list loaded from `src/main/resources/config/brand-policy.json`), and prohibited product categories (loaded from the same file). On violation, the guardrail returns a structured `brand-violation` rejection containing the offending text, the rule it broke, and a rewrite suggestion. The agent loop must rewrite and retry within its 4-iteration budget. On rejection, the guardrail also calls `AdJobEntity.recordBrandSafetyRejection(violations)` so the rejection is visible in the UI's rejection-log strip and in the audit log.

- **G2 — `before-tool-call` guardrail (scraping policy)**: `ScrapingPolicyGuardrail` is registered on `AdCreatorAgent` and runs before every tool call. It reads the candidate tool call's target URL (carried as the first argument of `fetchProductPage` and `extractAttributes`) and the allow-list recorded on `AdJobEntity` at job-creation time. A URL not on the allow-list is rejected before the tool body runs. The rejection returns a structured `scraping-policy-violation` error to the agent loop and calls `AdJobEntity.recordScrapingRejection(url, reason)`. The same hook enforces phase order using the status-based matrix: SCRAPE tools require `status ∈ {CREATED, SCRAPING}`; COPY tools require `status ∈ {SCRAPED, COPYING}` AND `profile.isPresent()`; VISUAL tools require `status ∈ {COPIED, GENERATING_VISUAL}` AND `copy.isPresent()`.

## 9. Agent prompts

- `AdCreatorAgent` → `prompts/ad-creator-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. It also instructs the agent to avoid superlatives without evidence, avoid competitor brand names, and comply with brand policy on the first attempt to minimise guardrail rejections.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded product URL for `Zenith Wireless Headphones`; within 60 s the job reaches `EVALUATED` with a non-empty `ProductProfile`, ≥ 2 ad variants, a `VisualSpec`, and an eval score chip on the card.
2. **J2** — The agent's first COPY iteration emits ad copy containing the phrase "best headphones on the market" (disallowed superlative, mock LLM path). `BrandSafetyGuardrail` blocks the response; a `BrandSafetyRejected` event lands on the entity; the agent rewrites; the job eventually completes correctly. The UI's rejection-log strip shows the one blocked response.
3. **J3** — The agent attempts to call `fetchProductPage` with a URL outside the allow-list (mock LLM path). `ScrapingPolicyGuardrail` rejects the call; a `ScrapingRejected` event lands; the agent retries with the approved URL; the pipeline completes.
4. **J4** — An ad package whose copy contains no product attributes from the scraped `ProductProfile` is scored 1 with a rationale naming the disconnected copy; the UI flags the card.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ad-creator-pipeline demonstrating the sequential-pipeline x
sales-marketing cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-sales-marketing-ad-creator-pipeline.
Java package io.akka.samples.aiadsgenerator. Akka 3.6.0. HTTP port 9438.

Components to wire (exactly):

- 1 AutonomousAgent AdCreatorAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/ad-creator-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  SCRAPE, COPY, and VISUAL tool sets are ALL registered on the agent; phase gating is the
  job of ScrapingPolicyGuardrail, NOT of conditional .tools(...) wiring. Both guardrails
  (ScrapingPolicyGuardrail and BrandSafetyGuardrail) are registered on the agent via the
  agent's guardrail-configuration block. On guardrail rejection the agent loop retries
  within its 4-iteration budget.

- 1 Workflow AdPipelineWorkflow per adJobId with four steps:
  * scrapeStep — emits ScrapeStarted on the entity, then calls componentClient
    .forAutonomousAgent(AdCreatorAgent.class, "agent-" + adJobId).runSingleTask(
      TaskDef.instructions("Product URL: " + productUrl + "\nPhase: SCRAPE\nUse
      fetchProductPage and extractAttributes to build a ProductProfile.")
        .metadata("adJobId", adJobId)
        .metadata("phase", "SCRAPE")
        .taskType(AdTasks.SCRAPE_PRODUCT)
    ). Reads forTask(taskId).result(SCRAPE_PRODUCT) to get ProductProfile. Writes
    AdJobEntity.recordProfile(profile). WorkflowSettings.stepTimeout 60s.
  * copyStep — emits CopyStarted, then runSingleTask with TaskDef.instructions
    (formatCopyContext(profile, productUrl)) and metadata.phase = "COPY", taskType
    DRAFT_COPY. Writes AdJobEntity.recordCopy(copy). stepTimeout 60s.
  * visualStep — emits VisualStarted, then runSingleTask with TaskDef.instructions
    (formatVisualContext(copy, profile)) and metadata.phase = "VISUAL", taskType
    GENERATE_VISUAL. Writes AdJobEntity.recordVisual(visual). stepTimeout 60s.
  * evalStep — assembles AdPackage from (copy, visual, profile), runs the deterministic
    AdQualityScorer, and writes AdJobEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(AdPipelineWorkflow::error). The error step writes
  AdJobFailed and ends.

- 1 EventSourcedEntity AdJobEntity (one per adJobId). State AdJobRecord{adJobId,
  productUrl: Optional<String>, profile: Optional<ProductProfile>, copy: Optional<AdCopy>,
  visual: Optional<VisualSpec>, adPackage: Optional<AdPackage>, eval: Optional<EvalResult>,
  status: AdJobStatus, createdAt: Instant, finishedAt: Optional<Instant>}. AdJobStatus enum:
  CREATED, SCRAPING, SCRAPED, COPYING, COPIED, GENERATING_VISUAL, VISUAL_GENERATED,
  EVALUATED, FAILED. Events: AdJobCreated{productUrl}, ScrapeStarted, ProductScraped{profile},
  CopyStarted, CopyDrafted{copy}, VisualStarted, VisualGenerated{visual}, AdEvaluated{eval},
  ScrapingRejected{url, reason}, BrandSafetyRejected{violations: List<BrandViolation>},
  AdJobFailed{reason}. Commands: create, startScrape, recordProfile, startCopy, recordCopy,
  startVisual, recordVisual, recordEvaluation, recordScrapingRejection,
  recordBrandSafetyRejection, fail, getAdJob. emptyState() returns AdJobRecord.initial("")
  with all Optional fields as Optional.empty() and no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 View AdJobView with row type AdJobRow that mirrors AdJobRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes AdJobEntity events. ONE query
  getAllAdJobs: SELECT * AS adJobs FROM ad_job_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * AdJobEndpoint at /api with POST /ad-jobs (body {productUrl}; mints adJobId; calls
    AdJobEntity.create(productUrl); then starts AdPipelineWorkflow with id
    "pipeline-" + adJobId; returns {adJobId}), GET /ad-jobs (list from getAllAdJobs,
    sorted newest-first), GET /ad-jobs/{id} (one row), GET /ad-jobs/sse (Server-Sent
    Events forwarded from the view's stream-updates), and three /api/metadata/*
    endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- AdTasks.java declaring three Task<R> constants:
    SCRAPE_PRODUCT = Task.name("Scrape product").description("Fetch the product page and
      extract a structured ProductProfile").resultConformsTo(ProductProfile.class);
    DRAFT_COPY = Task.name("Draft copy").description("Write ad copy variants for each
      target format from the ProductProfile").resultConformsTo(AdCopy.class);
    GENERATE_VISUAL = Task.name("Generate visual").description("Build an image-generation
      prompt and select aspect ratio from the AdCopy and ProductProfile")
      .resultConformsTo(VisualSpec.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- AdPhase.java — enum {SCRAPE, COPY, VISUAL}. Each function-tool method is annotated with
  the constant phase (use a custom annotation if the SDK's @FunctionTool does not carry a
  phase field — the guardrail reads it from a parallel registry built at startup if so).

- ScrapeTools.java — @FunctionTool fetchProductPage(String url) -> String reading the raw
  product HTML from src/main/resources/sample-data/products/<slug>.json's htmlFixture field;
  @FunctionTool extractAttributes(String html) -> ProductProfile parsing the HTML fixture
  into a structured ProductProfile using the same JSON file.

- CopyTools.java — @FunctionTool draftHeadline(ProductProfile profile) -> String producing
  a headline from the profile's productName and top attribute; @FunctionTool
  draftBody(ProductProfile profile, String format) -> AdVariant building one ad variant
  for the given format (INSTAGRAM / FACEBOOK / SEARCH) from the profile.

- VisualTools.java — @FunctionTool buildImagePrompt(ProductProfile profile, AdCopy copy) ->
  String composing a detailed Stable-Diffusion-style prompt from the product's visual
  attributes; @FunctionTool selectAspectRatio(String format) -> String returning the
  canonical ratio for the format (1:1 for INSTAGRAM, 4:3 for FACEBOOK, 3:1 for SEARCH).

- ScrapingPolicyGuardrail.java — implements the before-tool-call hook. Reads the candidate
  tool call's first String argument (URL), looks up AdJobEntity by adJobId (carried in the
  TaskDef metadata), checks the URL against the allow-list AND the phase-status matrix, and
  either passes or returns Guardrail.reject("scraping-policy-violation: ..."). On reject
  ALSO calls AdJobEntity.recordScrapingRejection(url, reason).

- BrandSafetyGuardrail.java — implements the before-agent-response hook. Scans the response
  text using the brand policy loaded from src/main/resources/config/brand-policy.json:
  disallowed_superlatives (list of strings), competitor_names (list), prohibited_categories
  (list). Returns Guardrail.accept() if clean, or Guardrail.reject(List<BrandViolation>)
  naming each rule breach and a rewrite suggestion. On reject ALSO calls
  AdJobEntity.recordBrandSafetyRejection(violations) so it appears in the UI rejection-log
  strip and the audit log.

- AdQualityScorer.java — pure deterministic logic (no LLM). Inputs: AdPackage, AdCopy,
  ProductProfile. Outputs: EvalResult with score and rationale. Four checks, one point per
  check satisfied, starting from a base of 1: attribute grounding (every AdVariant.body
  references at least one ProductAttribute.name from the ProductProfile), call-to-action
  present (AdCopy.callToAction is non-blank), visual-copy alignment (VisualSpec.imagePrompt
  references the productName), and variant coverage (AdCopy.variants.size() >= 2).
  Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9438 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/products.jsonl with 5 seeded product entries covering
  the three products named in J1-J4 plus two extras.

- src/main/resources/sample-data/products/*.json — three files keyed by product slug
  (zenith-wireless-headphones, alpine-trekking-boots, clearflow-water-bottle), each
  carrying a htmlFixture string and a pre-parsed ProductProfile with 5-8 attributes so
  ScrapeTools returns deterministic output across restarts.

- src/main/resources/config/brand-policy.json — disallowed_superlatives: ["best", "cheapest",
  "greatest", "perfect", "guaranteed"], competitor_names: ["RivalBrand", "CompeteCo"],
  prohibited_categories: ["pharmaceutical", "weapons", "gambling"].

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, G2) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  sample.

- risk-survey.yaml at the project root with data.data_classes.pii = false (product page data
  only), decisions.authority_level = recommend-only (the ad package is a draft for human
  review), oversight.human_in_loop = true (a marketer reviews before publishing),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "disallowed-claim-in-copy", "scraping-policy-violation",
  "missing-attribute-grounding", "phase-violation"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/ad-creator-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: AI Ads Generator", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of ad job cards; right = selected-job detail with product URL header, scraped
  attributes table, ad copy variants panel, visual spec panel, eval-score chip,
  rejection-log strip). Browser title exactly: <title>Akka Sample: AI Ads Generator</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(adJobId)), and
  deserialises into the task's typed return.
- Per-task mock-response structures for THIS blueprint:
    scrape-product.json — 5 ProductProfile entries, each with 5-8 ProductAttribute items.
      Each entry's tool_calls array contains: 1 fetchProductPage(url) + 1 extractAttributes(html).
      Plus 1 POLICY-VIOLATING entry whose tool_calls array starts with
      fetchProductPage("https://competitor.example.com/...") — ScrapingPolicyGuardrail rejects
      it, the mock then falls through to the approved URL. Selected on every 3rd job.
    draft-copy.json — 5 AdCopy entries paired with the scrape entries, each with 3 AdVariant
      items (INSTAGRAM, FACEBOOK, SEARCH). Plus 1 BRAND-VIOLATING entry whose primary
      headline starts with "Best wireless headphones guaranteed!" — BrandSafetyGuardrail
      blocks the response, mock replays a clean rewrite. Selected on every 2nd job.
    generate-visual.json — 5 VisualSpec entries, each with a detailed imagePrompt referencing
      the product name and key attributes; aspectRatio matching the primary variant format.
- MockModelProvider.seedFor(adJobId) makes per-job selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. AdCreatorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AdTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (scrapeStep
  60s, copyStep 60s, visualStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the AdJobRecord row record is Optional<T>.
- Lesson 7: AdTasks.java with SCRAPE_PRODUCT, DRAFT_COPY, GENERATE_VISUAL constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9438 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (AdCreatorAgent). The
  on-decision eval is rule-based (AdQualityScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  ScrapingPolicyGuardrail is the runtime mechanism that enforces phase order.
- Task dependency is carried by typed task results: scrapeStep writes ProductProfile onto
  the entity, copyStep reads it and builds the COPY task's instruction context from it,
  visualStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
