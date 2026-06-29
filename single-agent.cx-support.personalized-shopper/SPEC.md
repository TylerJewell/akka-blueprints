# SPEC — personalized-shopper

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Personalized Shopper.
**One-line pitch:** A shopper submits a preference profile; one AI agent reads the profile (passed as a task attachment, never as inline prompt text) against a product catalog and returns a structured recommendation set — ranked `ProductRecommendation` entries each with a rationale, a confidence tier, and an evidence reference to the catalog.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `ShoppingAdvisorAgent` (AutonomousAgent) carries the entire recommendation decision; the surrounding components only prepare its input and audit its output. One governance mechanism is wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw preference profile submission and the agent call — so the model never sees personal identifiers. Name, email address, phone number, loyalty-card number, and postal address are stripped before the profile reaches the LLM; only preference signals (categories, brands, price range, past-purchase categories, stated constraints) travel to the model.

The blueprint shows that even a straightforward recommendation use case requires explicit governance: shopper profiles are personal data, and routing raw profiles to an LLM call is a data-handling risk that a sanitizer prevents at the infrastructure level.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in a **preference profile** form: preferred categories (multi-select), preferred brands (free text, comma-separated), price range (min/max), excluded categories, and a free-text note ("I prefer sustainable materials"). Alternatively, the user picks one of three seeded profiles (budget-conscious electronics buyer, mid-range apparel shopper, home-goods renovator).
2. The user picks a **product catalog snapshot** from a dropdown (electronics, apparel, home goods) or accepts the default mixed catalog.
3. The user clicks **Get recommendations**. The UI POSTs to `/api/sessions` and receives a `sessionId`.
4. The session card appears in the live list in `STARTED` state. Within ~1 s it transitions to `PROFILE_SANITIZED` — the PII categories stripped are listed in the card detail.
5. Within ~10–30 s, the workflow's `recommendStep` completes. The card transitions to `RECOMMENDING` then `RECOMMENDATIONS_READY`. The recommendation set appears: a top-level outcome badge (MATCHED / PARTIAL_MATCH / NO_MATCH), a short explanation paragraph, and a ranked list of `ProductRecommendation` entries (rank, productId, productName, category, price, rationale, confidence tier).
6. Within ~1 s of the recommendations, the `freshnessStep` finishes. The card shows a **freshness score** chip (1–5) noting how well the recommended products align with the catalog's in-stock and current-season signals.
7. The user can submit another profile; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ShoppingEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ShoppingSessionEntity`, `RecommendationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ShoppingSessionEntity` | `EventSourcedEntity` | Per-session lifecycle: started → sanitized → recommending → ready → scored. Source of truth. | `ShoppingEndpoint`, `ProfileSanitizer`, `RecommendationWorkflow` | `RecommendationView` |
| `ProfileSanitizer` | `Consumer` | Subscribes to `SessionStarted` events; strips PII from the preference profile; calls `ShoppingSessionEntity.attachSanitized`. | `ShoppingSessionEntity` events | `ShoppingSessionEntity` |
| `RecommendationWorkflow` | `Workflow` | One workflow per session. Steps: `awaitSanitizedStep` → `recommendStep` → `freshnessStep`. | started by `ProfileSanitizer` once sanitized event lands | `ShoppingAdvisorAgent`, `ShoppingSessionEntity` |
| `ShoppingAdvisorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the catalog snapshot as task instructions and the sanitized profile as a task attachment; returns `RecommendationSet`. | invoked by `RecommendationWorkflow` | returns recommendation set |
| `RecommendationView` | `View` | Read model: one row per session for the UI. | `ShoppingSessionEntity` events | `ShoppingEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PreferenceProfile(
    String sessionId,
    List<String> preferredCategories,
    List<String> preferredBrands,
    double minPrice,
    double maxPrice,
    List<String> excludedCategories,
    String freeTextNote,
    String shopperEmail,          // PII — stripped before LLM
    String shopperPhone,          // PII — stripped before LLM
    String loyaltyCardNumber,     // PII — stripped before LLM
    String shopperName,           // PII — stripped before LLM
    Instant submittedAt
) {}

record SanitizedProfile(
    List<String> preferredCategories,
    List<String> preferredBrands,
    double minPrice,
    double maxPrice,
    List<String> excludedCategories,
    String freeTextNote,          // PII stripped from free text
    List<String> piiCategoriesStripped
) {}

record CatalogProduct(
    String productId,
    String productName,
    String category,
    String brand,
    double price,
    boolean inStock,
    boolean currentSeason,
    List<String> tags
) {}

record ProductRecommendation(
    int rank,
    String productId,
    String productName,
    String category,
    double price,
    String rationale,
    ConfidenceTier confidence
) {}
enum ConfidenceTier { HIGH, MEDIUM, LOW }

enum RecommendationOutcome { MATCHED, PARTIAL_MATCH, NO_MATCH }

record RecommendationSet(
    RecommendationOutcome outcome,
    String explanation,
    List<ProductRecommendation> recommendations,
    Instant decidedAt
) {}

record FreshnessResult(
    int score,          // 1..5
    String rationale,
    Instant scoredAt
) {}

record ShoppingSession(
    String sessionId,
    Optional<PreferenceProfile> profile,
    Optional<SanitizedProfile> sanitized,
    Optional<RecommendationSet> recommendations,
    Optional<FreshnessResult> freshness,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus {
    STARTED, PROFILE_SANITIZED, RECOMMENDING, RECOMMENDATIONS_READY, FRESHNESS_SCORED, FAILED
}
```

Events on `ShoppingSessionEntity`: `SessionStarted`, `ProfileSanitized`, `RecommendationStarted`, `RecommendationsRecorded`, `FreshnessScored`, `SessionFailed`.

Every nullable lifecycle field on the `ShoppingSession` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ preferredCategories, preferredBrands, minPrice, maxPrice, excludedCategories, freeTextNote, shopperEmail, shopperPhone, loyaltyCardNumber, shopperName, catalogSnapshot }` → `{ sessionId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Personalized Shopper</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted sessions (status pill + outcome badge + age) and a right pane with the selected session's detail — preference summary, PII-stripped categories, recommendation list, and freshness-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `ProfileSanitizer` Consumer): strips shopper name, email address, phone number, loyalty-card number, and postal address from the raw preference profile before any LLM call. Also applies regex and heuristic passes over `freeTextNote` to catch identifiers embedded in free text. Records which categories were stripped.

## 9. Agent prompts

- `ShoppingAdvisorAgent` → `prompts/shopping-advisor.md`. The single decision-making LLM. System prompt instructs it to read the sanitized preference profile (attached), walk the product catalog (task instructions), and return one `ProductRecommendation` per matched product ranked by relevance, up to ten results.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Shopper submits the electronics seed profile; within 30 s the recommendation set appears with at least three ranked entries and a freshness score chip.
2. **J2** — A profile containing `alice@example.com` and loyalty number `LC-99871234` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-LOYALTY-CARD]`; the entity's `profile.shopperEmail` retains the raw value for audit.
3. **J3** — A profile whose preferred categories have zero matching catalog products receives `outcome = NO_MATCH` and a clear explanation; the UI renders a "No matches found" card rather than an empty list.
4. **J4** — A `FreshnessResult` with `score = 1` is surfaced on the session card with a distinct visual flag, distinguishing it from higher-confidence recommendation sets.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named personalized-shopper demonstrating the single-agent × cx-support cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-personalized-shopper. Java package
io.akka.samples.personalizedshopping. Akka 3.6.0. HTTP port 9219.

Components to wire (exactly):

- 1 AutonomousAgent ShoppingAdvisorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/shopping-advisor.md>) and
  .capability(TaskAcceptance.of(ShoppingTasks.RECOMMEND_PRODUCTS).maxIterationsPerTask(3)).
  The task receives the catalog snapshot as its instruction text (formatted product list)
  and the sanitized profile as a task ATTACHMENT named "profile.txt"
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the
  canonical call). Output: RecommendationSet{outcome: RecommendationOutcome
  (MATCHED/PARTIAL_MATCH/NO_MATCH), explanation: String, recommendations:
  List<ProductRecommendation>, decidedAt: Instant}.

- 1 Workflow RecommendationWorkflow per sessionId with three steps:
  * awaitSanitizedStep — polls ShoppingSessionEntity.getSession every 1s; on
    session.sanitized().isPresent() advances to recommendStep.
    WorkflowSettings.stepTimeout 15s.
  * recommendStep — emits RecommendationStarted, then calls componentClient.forAutonomousAgent(
    ShoppingAdvisorAgent.class, "advisor-" + sessionId).runSingleTask(
      TaskDef.instructions(formatCatalog(catalogSnapshot))
        .attachment("profile.txt", session.sanitized.toProfileText().getBytes())
    ) — returns a taskId, then forTask(taskId).result(ShoppingTasks.RECOMMEND_PRODUCTS)
    to fetch the recommendation set. On success calls
    ShoppingSessionEntity.recordRecommendations(set).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(RecommendationWorkflow::error).
  * freshnessStep — runs a deterministic rule-based FreshnessScorer (NOT an LLM call)
    over the recorded RecommendationSet: checks how many recommended products are
    currently inStock and currentSeason in the catalog snapshot, whether any out-of-stock
    products were ranked first, and whether the total recommendation count matches the
    profile's breadth. Emits FreshnessScored{score: 1-5, rationale: String}.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ShoppingSessionEntity (one per sessionId). State
  ShoppingSession{sessionId: String, profile: Optional<PreferenceProfile>,
  sanitized: Optional<SanitizedProfile>, recommendations: Optional<RecommendationSet>,
  freshness: Optional<FreshnessResult>, status: SessionStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. SessionStatus enum: STARTED, PROFILE_SANITIZED,
  RECOMMENDING, RECOMMENDATIONS_READY, FRESHNESS_SCORED, FAILED. Events:
  SessionStarted{profile}, ProfileSanitized{sanitized}, RecommendationStarted{},
  RecommendationsRecorded{recommendations}, FreshnessScored{freshness},
  SessionFailed{reason}. Commands: start, attachSanitized, markRecommending,
  recordRecommendations, recordFreshness, fail, getSession. emptyState() returns
  ShoppingSession.initial("") with no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer ProfileSanitizer subscribed to ShoppingSessionEntity events; on
  SessionStarted runs a regex+heuristic stripping pipeline (email, phone number,
  loyalty-card-like tokens, postal addresses, person-name heuristic) over the
  PreferenceProfile PII fields and freeTextNote, computes the list of categories
  stripped, builds SanitizedProfile retaining only preference signals, then calls
  ShoppingSessionEntity.attachSanitized(sanitized). After attachSanitized lands, the
  same Consumer starts a RecommendationWorkflow with id = "rec-" + sessionId.

- 1 View RecommendationView with row type SessionRow (mirrors ShoppingSession minus
  profile.shopperEmail, profile.shopperPhone, profile.loyaltyCardNumber,
  profile.shopperName — the audit log keeps those; the view holds sanitized signals
  for the UI). Table updater consumes ShoppingSessionEntity events. ONE query
  getAllSessions: SELECT * AS sessions FROM recommendation_view. No WHERE status
  filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ShoppingEndpoint at /api with POST /sessions (body
    {preferredCategories: [String], preferredBrands: [String], minPrice: double,
    maxPrice: double, excludedCategories: [String], freeTextNote: String,
    shopperEmail: String, shopperPhone: String, loyaltyCardNumber: String,
    shopperName: String, catalogSnapshot: String};
    mints sessionId; calls ShoppingSessionEntity.start; returns {sessionId}),
    GET /sessions (list from getAllSessions, sorted newest-first),
    GET /sessions/{id} (one row), GET /sessions/sse (Server-Sent Events forwarded
    from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- ShoppingTasks.java declaring one Task<R> constant: RECOMMEND_PRODUCTS =
  Task.name("Recommend products").description("Read the attached preference profile
  and the product catalog, then produce a ranked RecommendationSet")
  .resultConformsTo(RecommendationSet.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records PreferenceProfile, SanitizedProfile, CatalogProduct,
  ProductRecommendation, ConfidenceTier, RecommendationOutcome, RecommendationSet,
  FreshnessResult, ShoppingSession, SessionStatus.

- FreshnessScorer.java — pure deterministic logic (no LLM). Inputs: RecommendationSet
  and the List<CatalogProduct> catalog snapshot. Outputs: FreshnessResult. Scoring
  rubric: +1 per recommended product that is inStock, +1 if all top-3 products are
  currentSeason, -1 if rank-1 product is out of stock, capped 1–5. Scoring rubric
  documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9219 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  ShoppingAdvisorAgent.definition() binds the configured provider via the per-agent
  override pattern from the akka-context docs.

- src/main/resources/sample-events/profiles.jsonl with 3 seeded preference profiles:
  a budget-conscious electronics buyer (prefers brand names under $200), a mid-range
  apparel shopper (sustainable brands, size M, $50–$150), and a home-goods renovator
  (Scandinavian style, $30–$300). Each profile contains 2–3 PII strings so S1 has
  work to do.

- src/main/resources/sample-events/catalog.jsonl with 20 seeded CatalogProduct entries
  spanning 3 categories (electronics × 7, apparel × 7, home-goods × 6), mix of
  in-stock/out-of-stock and current-season/off-season.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (S1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = recommend-only
  (the agent's recommendations are advisory — the shopper decides what to buy),
  oversight.human_in_loop = true (a human makes the purchase decision),
  failure.failure_modes including "irrelevant-recommendation", "pii-leakage-via-llm",
  "out-of-stock-recommendation", "preference-signal-loss-in-sanitization";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/shopping-advisor.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Personalized Shopper",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of session cards; right = selected-session detail with preference
  summary, PII-stripped categories, recommendation list ranked, and freshness-score
  chip). Browser title exactly: <title>Akka Sample: Personalized Shopper</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id.
- Per-task mock-response shapes for THIS blueprint:
    recommend-products.json — 6 RecommendationSet entries covering all three
    RecommendationOutcome values (2 MATCHED, 2 PARTIAL_MATCH, 2 NO_MATCH). Each MATCHED
    entry has 5–8 ProductRecommendation items; each PARTIAL_MATCH has 2–3; each NO_MATCH
    has an empty list with an explanation. Confidence tiers vary across entries.
    The mock selects deterministically based on seedFor(sessionId) so results are
    reproducible across restarts.
- A MockModelProvider.seedFor(sessionId) helper makes per-session selection
  deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ShoppingAdvisorAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion ShoppingTasks.java
  MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (recommendStep 60s, awaitSanitizedStep 15s, freshnessStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the ShoppingSession row record is
  Optional<T>. The view table updater wraps values with Optional.of(...); callers use
  .orElse(...) or .isPresent().
- Lesson 7: ShoppingTasks.java with RECOMMEND_PRODUCTS = Task.name(...)
  .description(...).resultConformsTo(RecommendationSet.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9219 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent
  (ShoppingAdvisorAgent). The freshness scorer is rule-based (FreshnessScorer.java)
  and does NOT make an LLM call.
- The preference profile is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated recommendStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
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
