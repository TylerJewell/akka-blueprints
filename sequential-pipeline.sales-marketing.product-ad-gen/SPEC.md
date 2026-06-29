# SPEC — product-catalog-ad-generation

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Product Catalog Ad Generation.
**One-line pitch:** A user submits a product catalog entry; one `AdGeneratorAgent` walks it through three task phases — **ENRICH** the catalog record with missing attributes, **DRAFT** a multi-format advertisement, **REVIEW** the draft for brand and compliance fit — with each phase gated on the prior phase's recorded output and the draft rejected by a brand guardrail before it is accepted.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a sales and marketing domain. One `AdGeneratorAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the ENRICH task's typed output becomes the DRAFT task's instruction context; the DRAFT task's typed output becomes the REVIEW task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- A **`before-agent-response` guardrail** sits between the agent and the workflow after every DRAFT-phase response. It reads the candidate `AdDraft` and applies the brand-policy rule set: required brand elements must appear in the headline, prohibited words must be absent from all copy fields, the ad's call-to-action must end with a URL-safe slug, and the character counts for each placement type must be within the declared limits. A draft that fails any rule is rejected before `AdDrafted` is recorded. The rejection carries a structured list of violations so the agent can self-correct inside its 4-iteration budget. The same guardrail also fires on the REVIEW-phase response for the final `ProductAd`, giving a second pass before the job is marked complete.

The blueprint shows that in a sequential pipeline the `before-agent-response` hook is the right cut for structural brand compliance: the agent produces a full structured response, the guardrail inspects its prose and shape as a whole (not one tool call at a time), and the rejection is an atomic correction notice — not a mid-loop interruption.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **product** from the catalog picker (or types a free-form product name). The UI POSTs to `/api/ad-jobs` and receives a `jobId`.
2. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `ENRICHING` — the workflow has started `enrichStep` and the agent has been handed the ENRICH task.
3. Within ~10–20 s the card reaches `DRAFTING` — the typed `EnrichedProduct` is visible in the card detail (attribute table: name, category, key features, pricing tier, target audience). The agent's ENRICH task returned; the workflow recorded `ProductEnriched` and ran the DRAFT task.
4. Within ~10–20 s more the card reaches `REVIEWING`. The `AdDraft` is visible (headline, body copy, call-to-action, placement variants for search, display, and social).
5. Within ~10–20 s more the card reaches `SCORED`. The right pane shows the final `ProductAd` — headline, approved copy per placement, and CTA — plus a compliance score chip (1–5) and a one-line rationale.
6. The user can submit another product; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AdJobEndpoint` | `HttpEndpoint` | `/api/ad-jobs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `AdJobEntity`, `AdJobView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AdJobEntity` | `EventSourcedEntity` | Per-job lifecycle: created → enriching → enriched → drafting → drafted → reviewing → approved → scored. Source of truth. | `AdJobEndpoint`, `AdGenerationWorkflow` | `AdJobView` |
| `AdGenerationWorkflow` | `Workflow` | One workflow per jobId. Steps: `enrichStep` → `draftStep` → `reviewStep` → `complianceStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `AdJobEndpoint` after `CREATED` | `AdGeneratorAgent`, `AdJobEntity` |
| `AdGeneratorAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `AdTasks.java`: `ENRICH_PRODUCT` → `EnrichedProduct`, `DRAFT_AD` → `AdDraft`, `REVIEW_AD` → `ProductAd`. Each task is registered with the phase-appropriate function tools. | invoked by `AdGenerationWorkflow` | returns typed results |
| `EnrichTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `lookupCategory(productName)` and `inferAttributes(productName, category)`. Reads from `src/main/resources/sample-data/catalog/*.json` for deterministic offline output. | called from ENRICH task | returns `EnrichedProduct` |
| `DraftTools` | function-tools class | Implements `composeHeadline(product)` and `composeCopy(product, placement)`. In-memory rule-based templates, parameterised by placement type. | called from DRAFT task | returns `String` (headline or copy) |
| `ReviewTools` | function-tools class | Implements `checkBrandElements(draft)` and `finaliseAd(draft)`. | called from REVIEW task | returns `List<PolicyViolation>` / `ProductAd` |
| `BrandGuardrail` | `before-agent-response` guardrail (registered on `AdGeneratorAgent`) | Reads the agent's candidate response. On DRAFT or REVIEW phase responses, applies the brand-policy rule set. Rejects any draft/ad that fails rules, returning a structured `BrandRejection{violations}` to the agent loop. On accept, emits `GuardrailAccepted`. | every agent response on DRAFT and REVIEW tasks | accept / structured-reject |
| `BrandComplianceScorer` | plain class (no Akka primitive) | Pure deterministic compliance evaluator. Inputs: `ProductAd`, `AdDraft`, `EnrichedProduct`. Output: `ComplianceResult{score, rationale}`. | called from `complianceStep` | returns score |
| `AdJobView` | `View` | Read model: one row per ad job for the UI. | `AdJobEntity` events | `AdJobEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ProductAttributes(
    String category,
    String pricingTier,       // "budget" | "mid-range" | "premium"
    List<String> keyFeatures,
    String targetAudience
) {}

record EnrichedProduct(
    String productId,
    String productName,
    ProductAttributes attributes,
    Instant enrichedAt
) {}

record AdPlacement(String type, String copy, int charCount) {} // type: "search" | "display" | "social"

record AdDraft(
    String headline,
    String callToAction,
    List<AdPlacement> placements,
    Instant draftedAt
) {}

record PolicyViolation(String field, String rule, String found) {}

record ProductAd(
    String jobId,
    String headline,
    String callToAction,
    List<AdPlacement> placements,
    Instant approvedAt
) {}

record ComplianceResult(
    int score,            // 1..5
    String rationale,
    Instant scoredAt
) {}

record BrandRejection(
    List<PolicyViolation> violations,
    Instant rejectedAt
) {}

record AdJobRecord(
    String jobId,
    Optional<String> productName,
    Optional<EnrichedProduct> enrichedProduct,
    Optional<AdDraft> adDraft,
    Optional<ProductAd> productAd,
    Optional<ComplianceResult> complianceResult,
    AdJobStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AdJobStatus {
    CREATED, ENRICHING, ENRICHED, DRAFTING, DRAFTED,
    REVIEWING, APPROVED, SCORED, FAILED
}
```

Events on `AdJobEntity`: `AdJobCreated`, `EnrichStarted`, `ProductEnriched`, `DraftStarted`, `AdDrafted`, `ReviewStarted`, `AdApproved`, `ComplianceScored`, `GuardrailRejected`, `AdJobFailed`.

Every nullable lifecycle field on the `AdJobRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/ad-jobs` — body `{ productName }` → `{ jobId }`.
- `GET /api/ad-jobs` — list all ad jobs, newest-first.
- `GET /api/ad-jobs/{id}` — one ad job.
- `GET /api/ad-jobs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Product Catalog Ad Generation</title>`.

The App UI tab is a two-column layout: a left rail with the live list of ad jobs (status pill + product name + age) and a right pane with the selected job's detail — product name, enriched-attribute table, ad draft placements, final ad copy, compliance score chip, and a guardrail-rejection log strip if any brand-policy rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (brand compliance)**: `BrandGuardrail` is registered on `AdGeneratorAgent` and runs before every DRAFT-phase and REVIEW-phase agent response is accepted. It reads the candidate `AdDraft` or `ProductAd` and applies four brand-policy rules: (1) the headline must contain at least one approved brand element from the brand-elements registry (`src/main/resources/brand-policy/elements.json`); (2) no copy field may contain words from the prohibited-words list (`src/main/resources/brand-policy/prohibited-words.txt`); (3) the call-to-action must end with a URL-safe slug matching `[a-z0-9-]+`; (4) each `AdPlacement.charCount` must be within the declared limit for its type (search ≤ 130 chars, display ≤ 300 chars, social ≤ 280 chars). On reject, the guardrail returns a structured `BrandRejection{violations}` to the agent loop and the workflow records a `GuardrailRejected{phase, field, rule, found}` event for audit. The agent loop retries within its 4-iteration budget.

## 9. Agent prompts

- `AdGeneratorAgent` → `prompts/ad-generator-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. On receiving a `BrandRejection`, the agent must address every listed violation before re-submitting.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded product `Wireless Noise-Cancelling Headphones`; within 60 s the job reaches `SCORED` with non-empty enriched attributes, a headline containing a brand element, ≥ 2 ad placements, and a compliance score chip ≥ 4.
2. **J2** — The agent's DRAFT-phase response contains a prohibited word (mock LLM path). `BrandGuardrail` rejects it; a `GuardrailRejected` event lands on the entity; the agent retries with corrected copy; the job eventually completes with a compliant ad. The UI's rejection-log strip shows the one rejected response.
3. **J3** — A draft whose headline lacks any brand element is scored 2 (headline-element rule fails, CTA rule passes, both placement-count rules pass, no prohibited words); the UI flags the card.
4. **J4** — Each task's instructions and tool calls are visible in the per-job trace (logged at `INFO`); the ENRICH task's log shows only ENRICH-tool calls, the DRAFT task's log shows only DRAFT-tool calls, the REVIEW task's log shows only REVIEW-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named product-catalog-ad-generation demonstrating the sequential-pipeline x
sales-marketing cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-sales-marketing-product-ad-gen. Java package
io.akka.samples.productcatalogadgeneration. Akka 3.6.0. HTTP port 9343.

Components to wire (exactly):

- 1 AutonomousAgent AdGeneratorAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/ad-generator-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(...) — the ENRICH, DRAFT, and REVIEW tool sets are ALL registered on the agent;
  the before-agent-response guardrail (BrandGuardrail) is the brand-compliance gate, NOT
  conditional .tools(...) wiring. The before-agent-response guardrail (BrandGuardrail) is
  registered on the agent via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow AdGenerationWorkflow per jobId with four steps:
  * enrichStep — emits EnrichStarted on the entity, then calls componentClient
    .forAutonomousAgent(AdGeneratorAgent.class, "agent-" + jobId).runSingleTask(
      TaskDef.instructions("Product name: " + productName + "\nPhase: ENRICH\nUse lookup
      and infer tools to enrich the product record.")
        .metadata("jobId", jobId)
        .metadata("phase", "ENRICH")
        .taskType(AdTasks.ENRICH_PRODUCT)
    ). Reads forTask(taskId).result(ENRICH_PRODUCT) to get EnrichedProduct. Writes
    AdJobEntity.recordEnrichment(enrichedProduct). WorkflowSettings.stepTimeout 60s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(enrichedProduct, productName)) and metadata.phase = "DRAFT", taskType
    DRAFT_AD. Writes AdJobEntity.recordDraft(adDraft). stepTimeout 60s.
  * reviewStep — emits ReviewStarted, then runSingleTask with TaskDef.instructions
    (formatReviewContext(adDraft, enrichedProduct, productName)) and metadata.phase = "REVIEW",
    taskType REVIEW_AD. Writes AdJobEntity.recordAd(productAd). stepTimeout 60s.
  * complianceStep — runs the deterministic BrandComplianceScorer over (productAd, adDraft,
    enrichedProduct) and writes AdJobEntity.recordCompliance(complianceResult). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(AdGenerationWorkflow::error). The error step writes
  AdJobFailed and ends.

- 1 EventSourcedEntity AdJobEntity (one per jobId). State AdJobRecord{jobId,
  productName: Optional<String>, enrichedProduct: Optional<EnrichedProduct>,
  adDraft: Optional<AdDraft>, productAd: Optional<ProductAd>,
  complianceResult: Optional<ComplianceResult>, status: AdJobStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. AdJobStatus enum: CREATED, ENRICHING, ENRICHED, DRAFTING,
  DRAFTED, REVIEWING, APPROVED, SCORED, FAILED. Events:
  AdJobCreated{productName}, EnrichStarted, ProductEnriched{enrichedProduct}, DraftStarted,
  AdDrafted{adDraft}, ReviewStarted, AdApproved{productAd}, ComplianceScored{complianceResult},
  GuardrailRejected{phase, field, rule, found}, AdJobFailed{reason}.
  Commands: create, startEnrich, recordEnrichment, startDraft, recordDraft, startReview,
  recordAd, recordCompliance, recordGuardrailRejection, fail, getAdJob. emptyState()
  returns AdJobRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View AdJobView with row type AdJobRow that mirrors AdJobRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes AdJobEntity events. ONE
  query getAllAdJobs: SELECT * AS adJobs FROM ad_job_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * AdJobEndpoint at /api with POST /ad-jobs (body {productName}; mints jobId; calls
    AdJobEntity.create(productName); then starts AdGenerationWorkflow with id
    "pipeline-" + jobId; returns {jobId}), GET /ad-jobs (list from
    getAllAdJobs, sorted newest-first), GET /ad-jobs/{id} (one row), GET
    /ad-jobs/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- AdTasks.java declaring three Task<R> constants:
    ENRICH_PRODUCT = Task.name("Enrich product").description("Look up category and infer
      attributes for the product to build a complete EnrichedProduct record")
      .resultConformsTo(EnrichedProduct.class);
    DRAFT_AD = Task.name("Draft ad").description("Compose headline, call-to-action, and
      placement copy for each ad placement type").resultConformsTo(AdDraft.class);
    REVIEW_AD = Task.name("Review ad").description("Check the draft, resolve any remaining
      brand issues, and produce the final ProductAd").resultConformsTo(ProductAd.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {ENRICH, DRAFT, REVIEW}. Each function-tool method is annotated with
  the constant phase (use a custom annotation if the SDK's @FunctionTool does not carry a
  phase field — the guardrail reads it from a parallel registry built at startup if so).

- EnrichTools.java — @FunctionTool lookupCategory(String productName) -> String reading from
  src/main/resources/sample-data/catalog/*.json keyed by productName slug; @FunctionTool
  inferAttributes(String productName, String category) -> ProductAttributes reading from
  the matching catalog entry.

- DraftTools.java — @FunctionTool composeHeadline(EnrichedProduct product) -> String
  (template-based, inserts product name + top key feature); @FunctionTool composeCopy(
  EnrichedProduct product, String placement) -> AdPlacement (reads char limits from
  src/main/resources/brand-policy/placement-limits.json and trims copy to fit).

- ReviewTools.java — @FunctionTool checkBrandElements(AdDraft draft) -> List<PolicyViolation>
  (reads elements.json and prohibited-words.txt; returns empty list on pass); @FunctionTool
  finaliseAd(AdDraft draft) -> ProductAd (assembles the ProductAd record, stamping
  approvedAt to Instant.now()).

- BrandGuardrail.java — implements the before-agent-response hook. Fires on DRAFT and REVIEW
  task responses. Reads the candidate AdDraft or ProductAd. Applies four brand-policy rules
  (headline element, prohibited words, CTA slug format, placement char limits). On any
  violation returns Guardrail.reject(BrandRejection{violations, rejectedAt}) AND calls
  AdJobEntity.recordGuardrailRejection(phase, field, rule, found) so the rejection appears
  in the UI's rejection-log strip and in the audit log. On full pass, returns
  Guardrail.accept().

- BrandComplianceScorer.java — pure deterministic logic (no LLM). Inputs: ProductAd,
  AdDraft, EnrichedProduct. Outputs: ComplianceResult with score and rationale. Four checks,
  one point per check satisfied, starting from a base of 1: headline element present (at
  least one brand element from elements.json appears in ProductAd.headline), no prohibited
  words (none of prohibited-words.txt appears in any copy field), CTA slug valid
  (callToAction ends with [a-z0-9-]+ regex), and placement parity
  (ProductAd.placements.size() >= 2). Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9343 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/catalog/*.json — four files keyed by product slug, each
  carrying a full catalog entry with category, pricingTier, keyFeatures list, and
  targetAudience field so EnrichTools returns the same record across restarts.

- src/main/resources/brand-policy/elements.json — list of approved brand elements
  (e.g. "ClearSound", "TrueComfort", "ProGrade") that the headline rule checks against.

- src/main/resources/brand-policy/prohibited-words.txt — one word per line; BrandGuardrail
  and BrandComplianceScorer both read this file.

- src/main/resources/brand-policy/placement-limits.json — map from placement type to
  char-count limit: search: 130, display: 300, social: 280.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  sales-marketing domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (product catalog
  records contain no personal data), decisions.authority_level = recommend-only (the ad is
  reviewed by a human before placement), oversight.human_in_loop = true (marketing team
  reviews every scored ad before it runs), operations.agent_count = 1,
  operations.agent_pattern = sequential-pipeline, failure.failure_modes including
  "prohibited-word-in-copy", "missing-brand-element", "cta-slug-invalid",
  "placement-over-char-limit", "enrichment-missing-attribute"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/ad-generator-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Product Catalog Ad Generation",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of ad job cards; right = selected-job detail with product name header,
  enriched-attribute table, ad draft placements, final ad copy per placement, compliance-score
  chip, rejection-log strip). Browser title exactly:
  <title>Akka Sample: Product Catalog Ad Generation</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below).
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(jobId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    enrich-product.json — 4 EnrichedProduct entries, one per seeded catalog product.
      Each entry's tool_calls array contains lookupCategory + inferAttributes calls.
    draft-ad.json — 4 AdDraft entries paired one-to-one with the enrich entries.
      Each entry's tool_calls contain composeHeadline + composeCopy (one per placement).
      Plus 1 deliberately VIOLATING entry whose headline contains a prohibited word —
      BrandGuardrail rejects it; the mock then falls through to a clean draft. The
      violating entry is selected on the FIRST iteration of every 3rd job (modulo seed)
      so J2 is reproducible.
    review-ad.json — 4 ProductAd entries paired one-to-one. Each carries 2-3 AdPlacement
      items matching the paired draft, with tool_calls containing checkBrandElements +
      finaliseAd. Plus 1 entry whose headline lacks any brand element — scored 2 by
      BrandComplianceScorer; J3 verifies this.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. AdGeneratorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AdTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (enrichStep
  60s, draftStep 60s, reviewStep 60s, complianceStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on AdJobRecord is Optional<T>. The view table
  updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: AdTasks.java with ENRICH_PRODUCT, DRAFT_AD, REVIEW_AD constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9343 declared explicitly in application.conf's
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
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (AdGeneratorAgent). The
  compliance eval is rule-based (BrandComplianceScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent; the
  before-agent-response guardrail (BrandGuardrail) is the runtime compliance gate.
- Task dependency is carried by typed task results: enrichStep writes EnrichedProduct onto
  the entity, draftStep reads it and builds the DRAFT task's instruction context, reviewStep
  reads both. The agent itself is stateless across phases.
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
