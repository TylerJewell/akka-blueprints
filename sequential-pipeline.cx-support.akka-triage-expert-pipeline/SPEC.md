# SPEC — triage-expert-multi-agent-workflow

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Triage + Expert Multi-Agent Workflow.
**One-line pitch:** A customer submits a support issue; a `TriageAgent` gathers their information and summarizes the problem, then a `SupportCaseWorkflow` scrubs PII from the summary and hands it to an `ExpertAgent`, which uses it to compose a grounded, guardrail-reviewed recommendation — with the typed handoff between agents as the only path information travels between phases.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a customer-support domain. Two distinct AutonomousAgents — `TriageAgent` and `ExpertAgent` — handle separate phases of the pipeline. The pattern's defining property is the **typed agent handoff**: `TriageAgent`'s output (`IssueSummary`) becomes `ExpertAgent`'s instruction context; the expert agent never sees the raw customer intake form. A `PiiSanitizer` runs between the two agents as an explicit sanitize step in the workflow, ensuring customer names, email addresses, account IDs, and phone numbers do not cross the agent boundary in the clear.

Three governance mechanisms are wired around the pipeline:

- A **`before-agent-response` guardrail** (`RecommendationGuardrail`) is registered on `ExpertAgent`. Before the typed `Recommendation` is written onto the entity and surfaced to the customer, the guardrail checks the composed text against a disallowed-content policy (no diagnostic promises, no price quotes, no competitor references). A blocked recommendation returns a structured reason to the agent loop; the agent retries inside its 3-iteration budget. The `GuardrailBlocked` event records every block for audit.
- A **PII sanitizer** (`PiiSanitizer`) runs as `sanitizeStep` in the workflow between triage and expert phases. It processes the `IssueSummary` produced by `TriageAgent` and replaces personal-data tokens with typed placeholders (`[CUSTOMER_NAME]`, `[ACCOUNT_ID]`, `[EMAIL]`, `[PHONE]`). The sanitized summary is what `ExpertAgent` receives; the original unsanitized summary stays on the entity for audit but never leaves the service perimeter.
- An **`on-decision-eval`** (`RecommendationScorer`) runs immediately after `RecommendationComposed` lands. It checks that every recommendation cites at least one knowledge-base article ID, that the article IDs cited exist in the known article corpus, and that the recommendation addresses the issue category declared in the triage summary. Emits a 1–5 grounding score.

## 3. User-facing flows

The user opens the App UI tab.

1. The customer types their **issue description** into the input (or picks one of three seeded categories — `Billing dispute`, `Account access locked`, `Product feature not working`).
2. The customer clicks **Submit issue**. The UI POSTs to `/api/cases` and receives a `caseId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `GATHERING` — the workflow has started `gatherStep` and `TriageAgent` is running the `GATHER_CUSTOMER_INFO` task.
4. Within ~10–20 s the card reaches `SUMMARIZING`. The typed `CustomerInfo` is visible in the card detail (name, account ID, issue category, urgency). The triage agent's gather task returned; the workflow recorded `CustomerInfoGathered` and is now running `SUMMARIZE_ISSUE`.
5. Within ~10–20 s more the card reaches `SANITIZING`. The `IssueSummary` is visible (category, problem statement, urgency, key facts). `PiiSanitizer` is running in-process.
6. Within ~1–2 s the card reaches `RECOMMENDING`. The sanitized summary is visible alongside a `[PII scrubbed]` badge. The `ExpertAgent` has been handed the `COMPOSE_RECOMMENDATION` task.
7. Within ~10–20 s more the card reaches `EVALUATED`. The right pane shows the full `Recommendation` — category, guidance text, cited knowledge-base articles — plus an eval score chip (1–5) and a one-line rationale.
8. The customer can submit another issue; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SupportCaseEndpoint` | `HttpEndpoint` | `/api/cases/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `SupportCaseEntity`, `SupportCaseView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SupportCaseEntity` | `EventSourcedEntity` | Per-case lifecycle: created → gathering → gathered → summarizing → summarized → sanitizing → sanitized → recommending → recommended → evaluated. Source of truth. | `SupportCaseEndpoint`, `SupportCaseWorkflow` | `SupportCaseView` |
| `SupportCaseWorkflow` | `Workflow` | One workflow per case. Steps: `gatherStep → summarizeStep → sanitizeStep → recommendStep → evalStep`. Each agent-calling step runs one task and writes the typed result back onto the entity before advancing. | started by `SupportCaseEndpoint` after `CREATED` | `TriageAgent`, `ExpertAgent`, `SupportCaseEntity` |
| `TriageAgent` | `AutonomousAgent` | The triage agent. Declares two `Task<R>` constants in `TriageTasks.java`: `GATHER_CUSTOMER_INFO` → `CustomerInfo`, `SUMMARIZE_ISSUE` → `IssueSummary`. Registered with `IntakeTools`. | invoked by `SupportCaseWorkflow` | returns typed results |
| `ExpertAgent` | `AutonomousAgent` | The expert agent. Declares one `Task<R>` constant in `ExpertTasks.java`: `COMPOSE_RECOMMENDATION` → `Recommendation`. Registered with `KnowledgeBaseTools`. The `before-agent-response` guardrail (`RecommendationGuardrail`) is registered on this agent. | invoked by `SupportCaseWorkflow` | returns typed result |
| `IntakeTools` | function-tools class | Implements `lookupCustomerAccount(accountId)` and `classifyIssue(description)`. Reads from `src/main/resources/sample-data/accounts/*.json` for deterministic offline output. | called from GATHER task | returns `AccountRecord` / `IssueCategory` |
| `KnowledgeBaseTools` | function-tools class | Implements `searchArticles(query)` and `fetchArticle(articleId)`. Reads from `src/main/resources/knowledge-base/articles/*.json`. | called from RECOMMEND task | returns `List<ArticleRef>` / `ArticleContent` |
| `PiiSanitizer` | plain class | Deterministic PII scrubber. Input: `IssueSummary`. Output: `SanitizedSummary`. Replaces name, email, account ID, phone tokens with typed placeholders. No LLM call. | called from `sanitizeStep` | returns `SanitizedSummary` |
| `RecommendationGuardrail` | `before-agent-response` guardrail (registered on `ExpertAgent`) | Checks the composed `Recommendation.guidanceText` against a disallowed-content policy: no diagnostic promises, no price quotes, no competitor references. On block, returns a structured rejection to the agent loop and records `GuardrailBlocked`. On pass, the result proceeds. | `ExpertAgent` response | accept / structured-block |
| `RecommendationScorer` | plain class | Deterministic on-decision evaluator. Inputs: `Recommendation`, `IssueSummary`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `SupportCaseView` | `View` | Read model: one row per case for the UI. | `SupportCaseEntity` events | `SupportCaseEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CustomerInfo(
    String customerId,
    String name,
    String email,
    String accountId,
    String phone,
    String issueDescription,
    IssueCategory category,
    UrgencyLevel urgency,
    Instant gatheredAt
) {}

enum IssueCategory {
    BILLING_DISPUTE, ACCOUNT_ACCESS, PRODUCT_FEATURE, OTHER
}

enum UrgencyLevel {
    LOW, MEDIUM, HIGH, CRITICAL
}

record IssueSummary(
    String caseId,
    IssueCategory category,
    UrgencyLevel urgency,
    String problemStatement,
    List<String> keyFacts,
    Instant summarizedAt
) {}

record SanitizedSummary(
    String caseId,
    IssueCategory category,
    UrgencyLevel urgency,
    String problemStatement,     // PII tokens replaced with placeholders
    List<String> keyFacts,       // PII tokens replaced
    List<String> redactedTokens, // Audit trail of what was scrubbed
    Instant sanitizedAt
) {}

record ArticleRef(String articleId, String title, double relevanceScore) {}

record ArticleContent(String articleId, String title, String body) {}

record Recommendation(
    String caseId,
    IssueCategory category,
    String guidanceText,
    List<ArticleRef> citedArticles,
    boolean requiresHumanEscalation,
    Instant composedAt
) {}

record EvalResult(
    int score,             // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record SupportCaseRecord(
    String caseId,
    Optional<String> issueDescription,
    Optional<CustomerInfo> customerInfo,
    Optional<IssueSummary> issueSummary,
    Optional<SanitizedSummary> sanitizedSummary,
    Optional<Recommendation> recommendation,
    Optional<EvalResult> eval,
    SupportCaseStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SupportCaseStatus {
    CREATED, GATHERING, GATHERED, SUMMARIZING, SUMMARIZED,
    SANITIZING, SANITIZED, RECOMMENDING, RECOMMENDED, EVALUATED, FAILED
}
```

Events on `SupportCaseEntity`: `CaseCreated`, `GatherStarted`, `CustomerInfoGathered`, `SummarizeStarted`, `IssueSummarized`, `SanitizeStarted`, `SummarySanitized`, `RecommendStarted`, `RecommendationComposed`, `EvaluationScored`, `GuardrailBlocked`, `CaseFailed`.

Every nullable lifecycle field on the `SupportCaseRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/cases` — body `{ issueDescription }` → `{ caseId }`.
- `GET /api/cases` — list all cases, newest-first.
- `GET /api/cases/{id}` — one case.
- `GET /api/cases/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Triage + Expert Multi-Agent Workflow</title>`.

The App UI tab is a two-column layout: a left rail with the live list of cases (status pill + issue description + age) and a right pane with the selected case's detail — customer info (redacted after sanitization), triage summary, sanitized summary with PII-scrub audit strip, recommendation with cited articles, eval score chip, and a guardrail-block log strip if any response blocks occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail**: `RecommendationGuardrail` is registered on `ExpertAgent` and runs before the agent's typed `Recommendation` is written onto the entity. It checks `Recommendation.guidanceText` against three disallowed-content rules: (1) no diagnostic promise (text matching "will fix", "guaranteed to resolve", "your issue is definitely" is blocked); (2) no price quote (text matching currency patterns or "free of charge" is blocked); (3) no competitor reference (a configurable list of competitor names is checked). On block, the guardrail returns a structured `policy-violation: <rule>` reason to the agent loop and calls `SupportCaseEntity.recordGuardrailBlock(rule, excerpt)` so the block is visible in the UI's guardrail-block log and in the audit log. The agent loop retries within its 3-iteration budget.
- **S1 — PII sanitizer**: `PiiSanitizer` runs as a standalone `sanitizeStep` in `SupportCaseWorkflow`, between `summarizeStep` and `recommendStep`. It applies four scrub passes (name, email, account ID, phone) to the `IssueSummary.problemStatement` and each item in `IssueSummary.keyFacts`, using deterministic regex patterns. The original `IssueSummary` stays on the entity (accessible to authorized audit paths); only the `SanitizedSummary` is forwarded to `ExpertAgent`. The redactedTokens list in `SanitizedSummary` records which fields were touched, without retaining the original values.
- **E1 — `on-decision-eval`**: runs immediately after `RecommendationComposed` lands, as `evalStep` in the workflow. `RecommendationScorer` is a deterministic rule-based scorer (no LLM call). Three checks, one point per check satisfied (base 1, max 4 → normalized to 1–5 after a readability weight): (1) article citation — `recommendation.citedArticles` is non-empty; (2) article provenance — every `ArticleRef.articleId` cited exists in the known article corpus (`src/main/resources/knowledge-base/articles/`); (3) category alignment — `recommendation.category` equals `sanitizedSummary.category`. A bonus point is awarded when `requiresHumanEscalation` is `true` and urgency is `CRITICAL` or `HIGH`. Emits `EvaluationScored{score:1..5, rationale}`.

## 9. Agent prompts

- `TriageAgent` → `prompts/triage-agent.md`. Handles two sequential tasks: gather customer info using intake tools, then summarize the issue into a structured `IssueSummary`. Never calls knowledge-base tools — those are registered only on `ExpertAgent`.
- `ExpertAgent` → `prompts/expert-agent.md`. Handles one task: compose a `Recommendation` grounded in knowledge-base articles. Receives the sanitized summary only; never sees raw customer PII. The `before-agent-response` guardrail reviews the output before it proceeds.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Customer submits `Billing dispute`; within 90 s the case reaches `EVALUATED` with non-empty `CustomerInfo`, an `IssueSummary`, a `SanitizedSummary` with redacted PII, a `Recommendation` citing ≥ 1 article, and an eval score chip.
2. **J2** — The mock LLM produces a recommendation containing a disallowed diagnostic promise. `RecommendationGuardrail` blocks it; a `GuardrailBlocked` event lands on the entity; the agent retries; the case eventually completes with a compliant recommendation. The UI's guardrail-block log strip shows the one blocked response.
3. **J3** — The sanitized summary passed to `ExpertAgent` contains none of the customer's original name, email, account ID, or phone number (all replaced by typed placeholders). Verified by asserting the original values do not appear in `SanitizedSummary.problemStatement` or `keyFacts`.
4. **J4** — A recommendation whose mock trajectory produces a citation to an unknown article ID is scored 1 with a rationale naming the unresolvable article; the UI flags the card.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named triage-expert-multi-agent-workflow demonstrating the
sequential-pipeline x cx-support cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-cx-support-akka-triage-expert-pipeline.
Java package io.akka.samples.triageexpertmultiagentworkflow. Akka 3.6.0.
HTTP port 9892.

Components to wire (exactly):

- 1 AutonomousAgent TriageAgent — extends
  akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded
  from prompts/triage-agent.md>) and two .capability(
  TaskAcceptance.of(TASK).maxIterationsPerTask(3)) entries — one per declared
  Task. Function tools: .tools(IntakeTools.class). No KnowledgeBaseTools here —
  tool separation is intentional (the expert agent has its own toolset).
  DO NOT register KnowledgeBaseTools on TriageAgent.

- 1 AutonomousAgent ExpertAgent — extends
  akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded
  from prompts/expert-agent.md>) and one .capability(
  TaskAcceptance.of(COMPOSE_RECOMMENDATION).maxIterationsPerTask(3)).
  Function tools: .tools(KnowledgeBaseTools.class). The before-agent-response
  guardrail (RecommendationGuardrail) is registered on this agent via the
  agent's guardrail-configuration block. On guardrail block the agent loop
  retries within its 3-iteration budget.

- 1 Workflow SupportCaseWorkflow per caseId with five steps:
  * gatherStep — emits GatherStarted on the entity, then calls componentClient
    .forAutonomousAgent(TriageAgent.class, "triage-" + caseId)
    .runSingleTask(TaskDef.instructions("Issue: " + issueDescription +
      "\nTask: GATHER_CUSTOMER_INFO\nUse lookupCustomerAccount and
      classifyIssue to gather structured customer info.")
        .metadata("caseId", caseId)
        .taskType(TriageTasks.GATHER_CUSTOMER_INFO)
    ). Reads result(GATHER_CUSTOMER_INFO) to get CustomerInfo. Writes
    SupportCaseEntity.recordCustomerInfo(customerInfo). stepTimeout 60s.
  * summarizeStep — emits SummarizeStarted, then runSingleTask with
    TaskDef.instructions(formatSummarizeContext(customerInfo, issueDescription))
    and metadata.caseId, taskType SUMMARIZE_ISSUE. Writes
    SupportCaseEntity.recordIssueSummary(summary). stepTimeout 60s.
  * sanitizeStep — emits SanitizeStarted, then calls PiiSanitizer.sanitize(
    issueSummary, customerInfo) in-process (no agent call). Writes
    SupportCaseEntity.recordSanitizedSummary(sanitized). stepTimeout 5s.
  * recommendStep — emits RecommendStarted, then calls componentClient
    .forAutonomousAgent(ExpertAgent.class, "expert-" + caseId)
    .runSingleTask(TaskDef.instructions(formatRecommendContext(
      sanitizedSummary))
        .metadata("caseId", caseId)
        .taskType(ExpertTasks.COMPOSE_RECOMMENDATION)
    ). Reads result(COMPOSE_RECOMMENDATION) to get Recommendation. The
    before-agent-response guardrail runs before the result is returned. Writes
    SupportCaseEntity.recordRecommendation(recommendation). stepTimeout 90s.
  * evalStep — runs RecommendationScorer.score(recommendation, sanitizedSummary)
    in-process and writes SupportCaseEntity.recordEvaluation(eval).
    stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a
  top-level WorkflowSettings class (Lesson 5). settings() declares
  defaultStepRecovery maxRetries(2).failoverTo(SupportCaseWorkflow::error).
  The error step writes CaseFailed and ends.

- 1 EventSourcedEntity SupportCaseEntity (one per caseId). State
  SupportCaseRecord{caseId, issueDescription: Optional<String>,
  customerInfo: Optional<CustomerInfo>, issueSummary: Optional<IssueSummary>,
  sanitizedSummary: Optional<SanitizedSummary>,
  recommendation: Optional<Recommendation>, eval: Optional<EvalResult>,
  status: SupportCaseStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}.
  SupportCaseStatus enum: CREATED, GATHERING, GATHERED, SUMMARIZING, SUMMARIZED,
  SANITIZING, SANITIZED, RECOMMENDING, RECOMMENDED, EVALUATED, FAILED.
  Events: CaseCreated{issueDescription}, GatherStarted, CustomerInfoGathered{
  customerInfo}, SummarizeStarted, IssueSummarized{issueSummary},
  SanitizeStarted, SummarySanitized{sanitizedSummary}, RecommendStarted,
  RecommendationComposed{recommendation}, EvaluationScored{eval},
  GuardrailBlocked{rule, excerpt, blockedAt}, CaseFailed{reason}.
  Commands: create, startGather, recordCustomerInfo, startSummarize,
  recordIssueSummary, startSanitize, recordSanitizedSummary, startRecommend,
  recordRecommendation, recordEvaluation, recordGuardrailBlock, fail, getCase.
  emptyState() returns SupportCaseRecord.initial("") with all Optional fields
  as Optional.empty() and no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 View SupportCaseView with row type SupportCaseRow that mirrors
  SupportCaseRecord exactly (all Optional<T> lifecycle fields preserved). Table
  updater consumes SupportCaseEntity events. ONE query getAllCases:
  SELECT * AS cases FROM support_case_view. No WHERE status filter — Akka
  cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SupportCaseEndpoint at /api with POST /cases (body {issueDescription};
    mints caseId; calls SupportCaseEntity.create(issueDescription); then starts
    SupportCaseWorkflow with id "workflow-" + caseId; returns {caseId}),
    GET /cases (list from getAllCases, sorted newest-first), GET /cases/{id}
    (one row), GET /cases/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion classes:

- TriageTasks.java declaring two Task<R> constants:
    GATHER_CUSTOMER_INFO = Task.name("Gather customer info").description(
      "Use lookupCustomerAccount and classifyIssue to collect structured
      customer and issue data").resultConformsTo(CustomerInfo.class);
    SUMMARIZE_ISSUE = Task.name("Summarize issue").description(
      "Distill the gathered customer info into a concise IssueSummary with
      category, urgency, problem statement, and key facts")
      .resultConformsTo(IssueSummary.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class
  (Lesson 7).

- ExpertTasks.java declaring one Task<R> constant:
    COMPOSE_RECOMMENDATION = Task.name("Compose recommendation").description(
      "Use knowledge-base tools to find relevant articles, then compose a
      grounded Recommendation citing them").resultConformsTo(Recommendation.class);
  DO NOT skip this (Lesson 7).

- IntakeTools.java — @FunctionTool lookupCustomerAccount(String accountId) ->
  AccountRecord reading from src/main/resources/sample-data/accounts/*.json
  keyed by accountId; @FunctionTool classifyIssue(String description) ->
  IssueCategory deterministic classification (BILLING_DISPUTE if description
  contains billing/invoice/charge/refund; ACCOUNT_ACCESS if
  locked/password/access; PRODUCT_FEATURE if feature/bug/not working; else
  OTHER).

- KnowledgeBaseTools.java — @FunctionTool searchArticles(String query) ->
  List<ArticleRef> reading from src/main/resources/knowledge-base/articles/
  index.json (fuzzy-match on title and tags, return top-5); @FunctionTool
  fetchArticle(String articleId) -> ArticleContent reading the matching JSON
  file.

- PiiSanitizer.java — pure deterministic logic (no LLM). Inputs: IssueSummary,
  CustomerInfo. Output: SanitizedSummary. Four regex scrub passes:
  (1) Replace CustomerInfo.name occurrences with [CUSTOMER_NAME].
  (2) Replace CustomerInfo.email occurrences with [EMAIL].
  (3) Replace CustomerInfo.accountId occurrences with [ACCOUNT_ID].
  (4) Replace CustomerInfo.phone occurrences with [PHONE].
  Apply to IssueSummary.problemStatement and each item in IssueSummary.keyFacts.
  Record each replaced token type in SanitizedSummary.redactedTokens (the type
  label only, not the original value). sanitizedAt = Instant.now().

- RecommendationGuardrail.java — implements the before-agent-response hook.
  Reads recommendation.guidanceText and checks three disallowed-content rules:
  (1) no-diagnostic-promise: text matches /will fix|guaranteed to resolve|your
  issue is definitely/i.
  (2) no-price-quote: text matches /\$\d|\d+\.\d\d|free of charge/i.
  (3) no-competitor-reference: text contains any name from a loaded configurable
  list (src/main/resources/config/competitors.txt).
  On any match: calls SupportCaseEntity.recordGuardrailBlock(rule, excerpt)
  and returns Guardrail.block("policy-violation: <rule>"). On all-pass: returns
  Guardrail.accept(). On block the agent loop retries within its 3-iteration
  budget.

- RecommendationScorer.java — pure deterministic logic (no LLM). Inputs:
  Recommendation, SanitizedSummary. Output: EvalResult. Three checks:
  (1) article citation — recommendation.citedArticles is non-empty (1 pt).
  (2) article provenance — every ArticleRef.articleId cited exists on disk in
  src/main/resources/knowledge-base/articles/ (1 pt).
  (3) category alignment — recommendation.category == sanitizedSummary.category
  (1 pt).
  Bonus point: requiresHumanEscalation == true AND urgency in {CRITICAL, HIGH}.
  Score = Math.min(1 + checks_passed + bonus, 5). Rationale names the first
  failing check, or "all checks passed" if score == 5.

- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9892 and the three model-provider blocks
  (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini gemini-2.5-flash)
  reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/issues.jsonl with 5 seeded issue description
  lines covering the three seeded categories plus two extras.

- src/main/resources/sample-data/accounts/*.json — three account files keyed by
  accountId, each carrying realistic but synthetic customer data
  (name, email, phone, accountId, accountType, accountStatus).

- src/main/resources/knowledge-base/articles/index.json and 6 article JSON files
  covering billing dispute resolution, account access recovery, product
  troubleshooting, escalation procedures, and refund policy.

- src/main/resources/config/competitors.txt — 10 placeholder competitor names
  used by RecommendationGuardrail rule (3).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, S1, E1) matching
  the mechanisms in Section 8 of this SPEC. Matching simplified_view list.
  No regulation_anchors — cx-support domain (deployer fills jurisdiction).

- risk-survey.yaml at the project root with data.data_classes.pii = true
  (customer name, email, account ID, phone collected during triage),
  decisions.authority_level = recommend-only (expert output is advisory),
  oversight.human_in_loop = true (a support agent reviews the recommendation
  before the customer is contacted), operations.agent_count = 2,
  operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "guardrail-blocked-recommendation",
  "pii-leak-across-agent-boundary", "uncited-recommendation",
  "category-mismatch", "guardrail-bypass-on-retry-exhaustion";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/triage-agent.md and prompts/expert-agent.md loaded as agent system
  prompts.

- README.md at the project root: title
  "Akka Sample: Triage + Expert Multi-Agent Workflow", prerequisites,
  generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO
  governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file
  (no ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses
  a two-column layout (left = live list of support case cards with status pill,
  issue category chip, age, and a red dot if any guardrail block fired; right =
  selected-case detail with customer info section, triage summary panel,
  sanitized-summary panel with PII-scrub audit strip, recommendation panel with
  cited articles, eval-score chip, guardrail-block log strip).
  Browser title exactly:
  <title>Akka Sample: Triage + Expert Multi-Agent Workflow</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        deterministic random-but-typed-correct outputs per Task (see Mock LLM
        provider block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing ModelProvider with per-task
  dispatch on Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry
  pseudo-randomly per call (seedFor(caseId)), deserialises into the task's
  typed return. Within a single task run the mock also drives tool-call
  sequences via each entry's "tool_calls" array.

- Per-task mock-response shapes:
    gather-customer-info.json — 5 CustomerInfo entries, each with a distinct
      IssueCategory, urgency mix, and tool_calls array containing 1
      lookupCustomerAccount call + 1 classifyIssue call.
    summarize-issue.json — 5 IssueSummary entries paired one-to-one with the
      gather entries, each with 2-4 keyFacts.
    compose-recommendation.json — 5 Recommendation entries paired one-to-one
      with the summarize entries, each citing 1-3 articleIds from the known
      corpus, with tool_calls containing searchArticles + 1-2 fetchArticle
      calls. Plus 1 deliberately BLOCKED entry whose guidanceText contains
      "will fix your issue" — the guardrail blocks it; the mock then falls
      through to a compliant recommendation. Select the blocking entry on the
      FIRST iteration of every 3rd case (modulo seed) so J2 is reproducible.
      Plus 1 deliberately UNCITED entry whose citedArticles list contains an
      articleId not present in the known corpus — scored 1 by the scorer; J4
      verifies this.

- A MockModelProvider.seedFor(caseId) helper makes per-case selection
  deterministic across restarts.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. TriageAgent and
  ExpertAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent. The
  companion TriageTasks.java and ExpertTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (gatherStep 60s, summarizeStep 60s, sanitizeStep 5s, recommendStep 90s,
  evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on SupportCaseRecord is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use
  .orElse(...) or .isPresent().
- Lesson 7: TriageTasks.java with GATHER_CUSTOMER_INFO and SUMMARIZE_ISSUE, and
  ExpertTasks.java with COMPOSE_RECOMMENDATION, are both mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9892 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in
  any user-visible string.
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND
  the mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. The DOM contains exactly five <section class="tab-panel">
  elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no
  more.
- The two-agent invariant: there are exactly TWO AutonomousAgents (TriageAgent,
  ExpertAgent). The PiiSanitizer and RecommendationScorer are pure logic
  classes — no LLM calls. The sequential-pipeline property holds: TriageAgent's
  output is the only input to ExpertAgent's task context.
- Tool separation: IntakeTools is registered ONLY on TriageAgent.
  KnowledgeBaseTools is registered ONLY on ExpertAgent. Do NOT cross-register.
- Task dependency is carried by typed task results: gatherStep writes
  CustomerInfo, summarizeStep reads it and builds the SUMMARIZE_ISSUE context,
  sanitizeStep reads IssueSummary and produces SanitizedSummary,
  recommendStep reads SanitizedSummary. No agent holds context from a previous
  agent's conversation.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export
  block (Lesson 25).
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
