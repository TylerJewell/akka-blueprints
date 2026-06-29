# SPEC — social-amplifier

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AI-Powered Social Media Amplifier.
**One-line pitch:** A user submits a source article; one `AmplifierAgent` walks it through three task phases — **PARSE** the article into key messages, **DRAFT** platform-tailored posts for each social channel, **PUBLISH** the approved posts — with each phase gated on the prior phase's recorded output, and brand-policy guardrails checking both the drafted content and the publish-intent before any write side-effects fire.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a sales-marketing domain. One `AmplifierAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the PARSE task's typed output becomes the DRAFT task's instruction context; the DRAFT task's typed output becomes the PUBLISH task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-agent-response` guardrail** intercepts the agent's drafted posts before the workflow records them. It applies a deterministic brand-policy ruleset to the full `DraftSet`: checks that no post exceeds the platform's character limit, that no post contains words from the brand-forbidden list, that each post includes a mandatory brand hashtag, and that posts targeting professional platforms do not include informal-tone markers. A policy violation returns a structured rejection to the agent loop so the DRAFT task can revise within its iteration budget. Brand-unsafe drafts never reach `DraftsProduced`; they are recorded as `BrandCheckFailed` events for audit.
- A **`before-tool-call` guardrail** sits between the agent and every publish tool. It reads the candidate tool call's target platform and the current `AmplificationEntity` status. A publish tool called while the entity has not yet recorded `DraftsProduced` (i.e., before brand-policy has passed) is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course. This is the second line of defence: even if a draft somehow bypassed the response-level check, no publish write fires until the entity's lifecycle records brand approval.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce both the dependency contract and the brand-safety perimeter.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes an **article URL** into the input (or picks one of three seeded articles — `Akka 3.6 launch post`, `Q3 developer survey results`, `AI regulation round-up June 2026`).
2. The user selects the **target platforms** from a checklist (LinkedIn, X, Bluesky; all selected by default).
3. The user clicks **Amplify**. The UI POSTs to `/api/amplifications` and receives an `amplificationId`.
4. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `PARSING` — the workflow has started `parseStep` and the agent has been handed the PARSE task.
5. Within ~10–20 s the card reaches `DRAFTING` — the typed `ParsedArticle` is visible in the card detail (headline, key messages, platform hints). The agent's PARSE task returned; the workflow recorded `ArticleParsed` and started the DRAFT task.
6. Within ~10–20 s more the card reaches `PUBLISHING`. The `DraftSet` is visible (one `Draft` per platform with text, hashtags, and a brand-check pass badge). Drafts that failed brand review at least once show a retry count.
7. Within ~10–20 s more the card reaches `PUBLISHED`. The right pane now shows the `PublishedSet` — one `PublicationReceipt` per platform (platform name, post URL stub, published timestamp) — plus a brand-audit score chip and a one-line rationale.
8. The user can submit another article; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AmplificationEndpoint` | `HttpEndpoint` | `/api/amplifications/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `AmplificationEntity`, `AmplificationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AmplificationEntity` | `EventSourcedEntity` | Per-run lifecycle: created → parsing → parsed → drafting → drafted → publishing → published. Source of truth. | `AmplificationEndpoint`, `AmplificationWorkflow` | `AmplificationView` |
| `AmplificationWorkflow` | `Workflow` | One workflow per amplificationId. Steps: `parseStep` → `draftStep` → `publishStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `AmplificationEndpoint` after `CREATED` | `AmplifierAgent`, `AmplificationEntity` |
| `AmplifierAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `AmplifierTasks.java`: `PARSE_ARTICLE` → `ParsedArticle`, `DRAFT_POSTS` → `DraftSet`, `PUBLISH_POSTS` → `PublishedSet`. Each task is registered with the phase-appropriate function tools. | invoked by `AmplificationWorkflow` | returns typed results |
| `ParseTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchArticle(url)` and `extractKeyMessages(article)`. Reads from `src/main/resources/sample-data/articles/*.json` for deterministic offline output. | called from PARSE task | returns `ParsedArticle` |
| `DraftTools` | function-tools class | Implements `draftPost(platform, keyMessages)` and `applyHashtags(draft, platform)`. Pure in-memory transformations. | called from DRAFT task | returns `Draft` |
| `PublishTools` | function-tools class | Implements `publishToLinkedIn(draft)`, `publishToX(draft)`, `publishToBluesky(draft)`. Write side-effects; stubbed in-process for offline use. | called from PUBLISH task | returns `PublicationReceipt` |
| `BrandPolicyGuardrail` | `before-agent-response` + `before-tool-call` guardrail (registered on `AmplifierAgent`) | Response hook: scores every `DraftSet` against brand-policy rules before the workflow records it. Tool-call hook: reads the candidate publish tool's target platform and the current `AmplificationEntity.status`; rejects publish calls until `DraftsProduced` is recorded. | every agent response on DRAFT task; every tool call on PUBLISH task | accept / structured-reject |
| `BrandPolicyScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `PublishedSet`, `DraftSet`, platform configs. Output: `BrandAuditResult{score, rationale}`. | called from `evalStep` | returns score |
| `AmplificationView` | `View` | Read model: one row per amplification run for the UI. | `AmplificationEntity` events | `AmplificationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record KeyMessage(String messageId, String text, String platformHint) {}

record ParsedArticle(
    String headline,
    String sourceUrl,
    List<KeyMessage> keyMessages,
    Instant parsedAt
) {}

record Draft(
    String draftId,
    Platform platform,
    String text,
    List<String> hashtags,
    int characterCount,
    int brandCheckAttempts
) {}

record DraftSet(
    List<Draft> drafts,
    Instant draftedAt
) {}

record PublicationReceipt(
    String receiptId,
    Platform platform,
    String postUrlStub,
    Instant publishedAt
) {}

record PublishedSet(
    List<PublicationReceipt> receipts,
    Instant publishedAt
) {}

record BrandAuditResult(
    int score,            // 1..5
    String rationale,
    Instant auditedAt
) {}

record BrandCheckRejection(
    String draftId,
    Platform platform,
    String rule,
    String reason,
    Instant rejectedAt
) {}

record AmplificationRecord(
    String amplificationId,
    Optional<String> sourceUrl,
    Optional<List<Platform>> targetPlatforms,
    Optional<ParsedArticle> parsedArticle,
    Optional<DraftSet> draftSet,
    Optional<PublishedSet> publishedSet,
    Optional<BrandAuditResult> brandAudit,
    AmplificationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<BrandCheckRejection> brandCheckRejections
) {}

enum AmplificationStatus {
    CREATED, PARSING, PARSED, DRAFTING, DRAFTED, PUBLISHING, PUBLISHED, FAILED
}

enum Platform {
    LINKEDIN, X, BLUESKY
}
```

Events on `AmplificationEntity`: `AmplificationCreated`, `ParseStarted`, `ArticleParsed`, `DraftStarted`, `DraftsProduced`, `PublishStarted`, `PostsPublished`, `BrandAuditCompleted`, `BrandCheckFailed`, `PublishToolBlocked`, `AmplificationFailed`.

Every nullable lifecycle field on the `AmplificationRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/amplifications` — body `{ sourceUrl, targetPlatforms }` → `{ amplificationId }`.
- `GET /api/amplifications` — list all runs, newest-first.
- `GET /api/amplifications/{id}` — one run.
- `GET /api/amplifications/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: AI-Powered Social Media Amplifier</title>`.

The App UI tab is a two-column layout: a left rail with the live list of amplification runs (status pill + source URL + age) and a right pane with the selected run's detail — article headline, key messages, draft posts per platform with brand-check status, publication receipts, brand-audit score chip, and a brand-check-rejection log strip if any policy violations occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (brand-policy check)**: `BrandPolicyGuardrail` is registered on `AmplifierAgent` and runs after every agent response on the DRAFT task. It applies four deterministic rules to the full `DraftSet`: (1) no post exceeds the platform's character limit (LinkedIn 3000, X 280, Bluesky 300), (2) no post contains a word from the brand-forbidden list (loaded from `src/main/resources/brand-policy/forbidden-words.txt`), (3) every post includes at least one mandatory brand hashtag (loaded from `src/main/resources/brand-policy/required-hashtags.txt`), (4) posts targeting LinkedIn and Bluesky contain no informal-tone markers. On reject, the guardrail returns a structured `brand-policy-violation` error specifying the failing rule and the offending draft to the agent loop; the workflow records a `BrandCheckFailed{draftId, platform, rule, reason}` event for audit. The agent loop retries within its 4-iteration budget.
- **G2 — `before-tool-call` guardrail (publish-intent gate)**: `BrandPolicyGuardrail` is also registered on the `before-tool-call` hook for the PUBLISH task. It reads the in-flight publish tool's target `Platform` and the current `AmplificationEntity.status`. The accept rule is: publish tools require `status ∈ {DRAFTED, PUBLISHING}` AND `draftSet.isPresent()`. On reject, the guardrail returns a structured `publish-gated` error to the agent loop and the workflow records a `PublishToolBlocked{platform, tool, reason}` event. The agent loop retries within its 4-iteration budget. This is the second line of defence: even if a draft passed brand review, the publish write cannot fire until the entity's lifecycle confirms brand approval.

## 9. Agent prompts

- `AmplifierAgent` → `prompts/amplifier-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded article `Akka 3.6 launch post`; within 60 s the run reaches `PUBLISHED` with a parsed article, ≥ 3 drafts (one per platform), all brand-check badges green, and a brand-audit score chip showing ≥ 4/5.
2. **J2** — The agent's first DRAFT iteration produces a post for X that exceeds the 280-character limit. `BrandPolicyGuardrail` (response hook) rejects the response; a `BrandCheckFailed` event lands on the entity; the agent retries with a shorter post; the run eventually completes. The UI's rejection-log strip shows the one failed brand check.
3. **J3** — The agent calls `publishToLinkedIn` before `DraftsProduced` has been recorded (mock LLM path). `BrandPolicyGuardrail` (tool-call hook) rejects the call; a `PublishToolBlocked` event lands on the entity; the agent retries the correct DRAFT sequence first; the run completes correctly.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-run trace (logged at `INFO`); the PARSE task's log shows only PARSE-tool calls, the DRAFT task's log shows only DRAFT-tool calls, the PUBLISH task's log shows only PUBLISH-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named social-amplifier demonstrating the sequential-pipeline x sales-marketing
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-sales-marketing-social-amplifier. Java package
io.akka.samples.aipoweredsocialmediaamplifier. Akka 3.6.0. HTTP port 9860.

Components to wire (exactly):

- 1 AutonomousAgent AmplifierAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/amplifier-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  PARSE, DRAFT, and PUBLISH tool sets are ALL registered on the agent; phase gating is the
  job of BrandPolicyGuardrail, NOT of conditional .tools(...) wiring. Both guardrail hooks
  (before-agent-response and before-tool-call) are registered on the agent via the agent's
  guardrail-configuration block.

- 1 Workflow AmplificationWorkflow per amplificationId with four steps:
  * parseStep — emits ParseStarted on the entity, then calls componentClient
    .forAutonomousAgent(AmplifierAgent.class, "agent-" + amplificationId).runSingleTask(
      TaskDef.instructions("Article URL: " + sourceUrl + "\nPlatforms: " + platforms +
      "\nPhase: PARSE\nFetch the article and extract 3-6 key messages with platform hints.")
        .metadata("amplificationId", amplificationId)
        .metadata("phase", "PARSE")
        .taskType(AmplifierTasks.PARSE_ARTICLE)
    ). Reads forTask(taskId).result(PARSE_ARTICLE) to get ParsedArticle. Writes
    AmplificationEntity.recordParsedArticle(parsedArticle). WorkflowSettings.stepTimeout 60s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(parsedArticle, targetPlatforms)) and metadata.phase = "DRAFT",
    taskType DRAFT_POSTS. Writes AmplificationEntity.recordDrafts(draftSet). stepTimeout 60s.
  * publishStep — emits PublishStarted, then runSingleTask with TaskDef.instructions
    (formatPublishContext(draftSet, parsedArticle)) and metadata.phase = "PUBLISH", taskType
    PUBLISH_POSTS. Writes AmplificationEntity.recordPublished(publishedSet). stepTimeout 60s.
  * evalStep — runs the deterministic BrandPolicyScorer over (publishedSet, draftSet,
    platformConfigs) and writes AmplificationEntity.recordBrandAudit(brandAudit).
    stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(AmplificationWorkflow::error). The error step writes
  AmplificationFailed and ends.

- 1 EventSourcedEntity AmplificationEntity (one per amplificationId). State
  AmplificationRecord{amplificationId, sourceUrl: Optional<String>,
  targetPlatforms: Optional<List<Platform>>, parsedArticle: Optional<ParsedArticle>,
  draftSet: Optional<DraftSet>, publishedSet: Optional<PublishedSet>,
  brandAudit: Optional<BrandAuditResult>, status: AmplificationStatus, createdAt: Instant,
  finishedAt: Optional<Instant>, brandCheckRejections: List<BrandCheckRejection>}.
  AmplificationStatus enum: CREATED, PARSING, PARSED, DRAFTING, DRAFTED, PUBLISHING,
  PUBLISHED, FAILED. Platform enum: LINKEDIN, X, BLUESKY. Events:
  AmplificationCreated{sourceUrl, targetPlatforms}, ParseStarted, ArticleParsed{parsedArticle},
  DraftStarted, DraftsProduced{draftSet}, PublishStarted, PostsPublished{publishedSet},
  BrandAuditCompleted{brandAudit}, BrandCheckFailed{draftId, platform, rule, reason,
  rejectedAt}, PublishToolBlocked{platform, tool, reason, rejectedAt},
  AmplificationFailed{reason}.
  Commands: create, startParse, recordParsedArticle, startDraft, recordDrafts, startPublish,
  recordPublished, recordBrandAudit, recordBrandCheckFailed, recordPublishToolBlocked, fail,
  getAmplification. emptyState() returns AmplificationRecord.initial("") with all Optional
  fields as Optional.empty() and no commandContext() reference (Lesson 3).

- 1 View AmplificationView with row type AmplificationRow that mirrors AmplificationRecord
  exactly (all Optional<T> lifecycle fields preserved). Table updater consumes
  AmplificationEntity events. ONE query getAllAmplifications: SELECT * AS amplifications FROM
  amplification_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * AmplificationEndpoint at /api with POST /amplifications (body {sourceUrl,
    targetPlatforms}; mints amplificationId; calls AmplificationEntity.create(sourceUrl,
    targetPlatforms); then starts AmplificationWorkflow with id "amp-" + amplificationId;
    returns {amplificationId}), GET /amplifications (list from getAllAmplifications,
    sorted newest-first), GET /amplifications/{id} (one row), GET /amplifications/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- AmplifierTasks.java declaring three Task<R> constants:
    PARSE_ARTICLE = Task.name("Parse article").description("Fetch the article and extract
      key messages with platform hints").resultConformsTo(ParsedArticle.class);
    DRAFT_POSTS = Task.name("Draft posts").description("Draft one platform-tailored post
      per target platform").resultConformsTo(DraftSet.class);
    PUBLISH_POSTS = Task.name("Publish posts").description("Publish each approved draft
      to its target platform and return receipts").resultConformsTo(PublishedSet.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- AmplifierPhase.java — enum {PARSE, DRAFT, PUBLISH}.

- ParseTools.java — @FunctionTool fetchArticle(String url) -> String reading from
  src/main/resources/sample-data/articles/*.json keyed by url slug; @FunctionTool
  extractKeyMessages(String articleText, List<String> targetPlatforms) -> List<KeyMessage>
  (3-6 messages with a platformHint per message derived from article content).

- DraftTools.java — @FunctionTool draftPost(String platform, List<KeyMessage> keyMessages,
  int charLimit) -> Draft (composes platform-tailored text from messages, respects charLimit);
  @FunctionTool applyHashtags(Draft draft, String platform) -> Draft (appends mandatory
  brand hashtags and removes duplicates within the platform's limit).

- PublishTools.java — @FunctionTool publishToLinkedIn(Draft draft) -> PublicationReceipt;
  @FunctionTool publishToX(Draft draft) -> PublicationReceipt;
  @FunctionTool publishToBluesky(Draft draft) -> PublicationReceipt.
  All three are in-process stubs that generate a deterministic postUrlStub
  ("https://stub.example.com/<platform>/<receipt-id>") and a publishedAt timestamp.

- BrandPolicyGuardrail.java — implements BOTH hooks.
  Response hook (before-agent-response on DRAFT task): receives the agent's DraftSet
  response, applies four rules per Draft:
    (1) characterCount <= platformCharLimit(platform)
    (2) no text token in brand-forbidden list
    (3) at least one required-hashtag present
    (4) no informal-tone markers on LINKEDIN / BLUESKY platforms
  On any violation: returns Guardrail.reject("brand-policy-violation: <rule>: <detail>") and
  calls AmplificationEntity.recordBrandCheckFailed(draftId, platform, rule, reason). On
  accept: passes the response through.
  Tool-call hook (before-tool-call on PUBLISH task): reads the candidate publish tool's
  Platform, looks up the AmplificationEntity status by amplificationId (carried in the
  TaskDef metadata), applies the accept matrix: publish tools require status ∈ {DRAFTED,
  PUBLISHING} AND draftSet.isPresent(). On reject: returns
  Guardrail.reject("publish-gated: <tool> requires DraftsProduced, saw <status>") and calls
  AmplificationEntity.recordPublishToolBlocked(platform, tool, reason). On accept: call
  proceeds.

- BrandPolicyScorer.java — pure deterministic logic (no LLM). Inputs: PublishedSet,
  DraftSet, List<Platform> configs. Outputs: BrandAuditResult with score and rationale. Four
  checks, one point per check satisfied, starting from a base of 1:
  (1) receipt coverage — every Draft in the DraftSet has a matching PublicationReceipt,
  (2) no platform receipt is a stub-failure (receipts with postUrlStub containing "error"),
  (3) all Drafts that reached PUBLISH had brandCheckAttempts <= 2 (no excessive retries),
  (4) every Draft.characterCount is within its platform's limit at publish time.
  Score range 1-5. Rationale names the largest gap or "all checks passed".

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9860 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/articles.jsonl with 5 seeded article lines covering the
  three surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/articles/*.json — three files keyed by seeded article slug,
  each carrying an article body and 6-10 KeyMessage entries with deterministic content so
  ParseTools.fetchArticle returns the same output across restarts.

- src/main/resources/brand-policy/forbidden-words.txt — newline-delimited list of 10-15
  brand-forbidden words for the guardrail to check.

- src/main/resources/brand-policy/required-hashtags.txt — newline-delimited list of 2-3
  mandatory brand hashtags (e.g., #AkkaBuilt, #AkkaSample).

- src/main/resources/brand-policy/platform-char-limits.json — JSON map
  {"LINKEDIN":3000,"X":280,"BLUESKY":300}.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, G2) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — sample
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (article text and
  drafts are not person-level data), decisions.authority_level = automated-action (posts are
  published without human approval in the default flow), oversight.human_in_loop = false,
  oversight.human_on_loop = true (a marketer reviews the brand-audit score after publish),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "brand-policy-violation", "publish-gated",
  "character-limit-exceeded", "missing-required-hashtag", "informal-tone-on-professional-platform";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/amplifier-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: AI-Powered Social Media Amplifier",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of amplification run cards; right = selected-run detail with article headline,
  parsed key messages, draft posts per platform with brand-check badges, publication receipts,
  brand-audit score chip, brand-check-rejection log strip). Browser title exactly:
  <title>Akka Sample: AI-Powered Social Media Amplifier</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(amplificationId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    parse-article.json — 5 ParsedArticle entries, each with 3-6 KeyMessage items per seeded
      article. Each entry's tool_calls array contains 1-2 calls: 1 fetchArticle(url) + 1
      extractKeyMessages(text, platforms) call.
    draft-posts.json — 5 DraftSet entries paired one-to-one with the parse entries, each
      with one Draft per platform, with tool_calls containing draftPost + applyHashtags per
      platform. Plus 1 deliberately BRAND-POLICY-VIOLATING entry whose X Draft exceeds 280
      characters — the guardrail rejects it, the mock then falls through to a corrected draft.
      The mock should select the violating entry on the FIRST iteration of every 3rd run
      (modulo seed) so J2 is reproducible.
    publish-posts.json — 5 PublishedSet entries paired one-to-one. Each carries one
      PublicationReceipt per platform, tool_calls containing publishToLinkedIn +
      publishToX + publishToBluesky. Plus 1 deliberately TOOL-GATED entry whose tool_calls
      array starts with publishToLinkedIn BEFORE draftSet is present — the guardrail rejects
      it; J3 verifies this.
- A MockModelProvider.seedFor(amplificationId) helper makes per-run selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. AmplifierAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AmplifierTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (parseStep
  60s, draftStep 60s, publishStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the AmplificationRecord row record is
  Optional<T>. The view table updater wraps values with Optional.of(...); callers use
  .orElse(...) or .isPresent().
- Lesson 7: AmplifierTasks.java with PARSE_ARTICLE, DRAFT_POSTS, PUBLISH_POSTS constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9860 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (AmplifierAgent). The
  on-decision eval is rule-based (BrandPolicyScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the brand-policy guardrail is the runtime mechanism that enforces both the content safety
  perimeter (response hook) and the phase order (tool-call hook). Do NOT conditionally
  register tools per task — the guardrail is the gate.
- Task dependency is carried by typed task results: parseStep writes ParsedArticle onto
  the entity, draftStep reads it and builds the DRAFT task's instruction context from it,
  publishStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
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
