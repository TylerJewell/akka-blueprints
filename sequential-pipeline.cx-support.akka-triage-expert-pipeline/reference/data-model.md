# Data model — triage-expert-multi-agent-workflow

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CustomerInfo` | `customerId` | `String` | no | System-assigned customer identifier. |
| | `name` | `String` | no | Customer's full name. PII — scrubbed by `PiiSanitizer` before crossing the agent boundary. |
| | `email` | `String` | no | Customer's email address. PII — scrubbed. |
| | `accountId` | `String` | no | Account identifier supplied in the issue description or looked up via `IntakeTools`. PII — scrubbed. |
| | `phone` | `String` | no | Customer's phone number. PII — scrubbed. |
| | `issueDescription` | `String` | no | Original free-text description submitted by the customer. |
| | `category` | `IssueCategory` | no | Classified by `IntakeTools.classifyIssue`. |
| | `urgency` | `UrgencyLevel` | no | Derived from issue description by `TriageAgent`. |
| | `gatheredAt` | `Instant` | no | When the GATHER task returned. |
| `IssueSummary` | `caseId` | `String` | no | Links back to the parent `SupportCaseEntity`. |
| | `category` | `IssueCategory` | no | Carried forward from `CustomerInfo`. |
| | `urgency` | `UrgencyLevel` | no | Carried forward from `CustomerInfo`. |
| | `problemStatement` | `String` | no | 1–2 sentences describing what the customer cannot do. May contain PII — scrubbed before forwarding. |
| | `keyFacts` | `List<String>` | no | 2–5 discrete facts. May contain PII — each item scrubbed. |
| | `summarizedAt` | `Instant` | no | When the SUMMARIZE task returned. |
| `SanitizedSummary` | `caseId` | `String` | no | Links to parent entity. |
| | `category` | `IssueCategory` | no | Unchanged from `IssueSummary`. |
| | `urgency` | `UrgencyLevel` | no | Unchanged from `IssueSummary`. |
| | `problemStatement` | `String` | no | PII tokens replaced with typed placeholders. |
| | `keyFacts` | `List<String>` | no | Each item scrubbed. |
| | `redactedTokens` | `List<String>` | no | Token type labels that were replaced (e.g., `"CUSTOMER_NAME"`). Does not retain original values. |
| | `sanitizedAt` | `Instant` | no | When `PiiSanitizer` completed. |
| `ArticleRef` | `articleId` | `String` | no | Knowledge-base article identifier. Must exist in `knowledge-base/articles/`. |
| | `title` | `String` | no | Article title. |
| | `relevanceScore` | `double` | no | 0.0–1.0 relevance to the search query. |
| `ArticleContent` | `articleId` | `String` | no | Article identifier. |
| | `title` | `String` | no | Article title. |
| | `body` | `String` | no | Full article text. |
| `Recommendation` | `caseId` | `String` | no | Links to parent entity. |
| | `category` | `IssueCategory` | no | MUST equal `sanitizedSummary.category` (E1 rule 3). |
| | `guidanceText` | `String` | no | 2–5 sentences of actionable guidance. Subject to guardrail policy checks. |
| | `citedArticles` | `List<ArticleRef>` | no | Non-empty (E1 rule 1). Every `articleId` must exist on disk (E1 rule 2). |
| | `requiresHumanEscalation` | `boolean` | no | `true` when urgency is HIGH/CRITICAL and no article fully resolves the issue. |
| | `composedAt` | `Instant` | no | When the COMPOSE_RECOMMENDATION task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the first failing check, or "all checks passed". |
| | `evaluatedAt` | `Instant` | no | When `RecommendationScorer` finished. |
| `GuardrailBlockRecord` | `rule` | `String` | no | Name of the violated disallowed-content rule. |
| | `excerpt` | `String` | no | Short excerpt of the offending text. |
| | `blockedAt` | `Instant` | no | When the guardrail fired. |
| `SupportCaseRecord` (entity state) | `caseId` | `String` | no | — |
| | `issueDescription` | `Optional<String>` | yes | Populated after `CaseCreated`. |
| | `customerInfo` | `Optional<CustomerInfo>` | yes | Populated after `CustomerInfoGathered`. |
| | `issueSummary` | `Optional<IssueSummary>` | yes | Populated after `IssueSummarized`. |
| | `sanitizedSummary` | `Optional<SanitizedSummary>` | yes | Populated after `SummarySanitized`. |
| | `recommendation` | `Optional<Recommendation>` | yes | Populated after `RecommendationComposed`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `SupportCaseStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `CaseCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailBlocks` | `List<GuardrailBlockRecord>` | no | Appended on every `GuardrailBlocked` event; empty on the happy path. |

Every nullable lifecycle field on `SupportCaseRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SupportCaseStatus`: `CREATED`, `GATHERING`, `GATHERED`, `SUMMARIZING`, `SUMMARIZED`, `SANITIZING`, `SANITIZED`, `RECOMMENDING`, `RECOMMENDED`, `EVALUATED`, `FAILED`.

`IssueCategory`: `BILLING_DISPUTE`, `ACCOUNT_ACCESS`, `PRODUCT_FEATURE`, `OTHER`.

`UrgencyLevel`: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.

## Events (`SupportCaseEntity`)

| Event | Payload | Transition |
|---|---|---|
| `CaseCreated` | `issueDescription: String` | → CREATED |
| `GatherStarted` | — | → GATHERING |
| `CustomerInfoGathered` | `customerInfo: CustomerInfo` | → GATHERED |
| `SummarizeStarted` | — | → SUMMARIZING |
| `IssueSummarized` | `issueSummary: IssueSummary` | → SUMMARIZED |
| `SanitizeStarted` | — | → SANITIZING |
| `SummarySanitized` | `sanitizedSummary: SanitizedSummary` | → SANITIZED |
| `RecommendStarted` | — | → RECOMMENDING |
| `RecommendationComposed` | `recommendation: Recommendation` | → RECOMMENDED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `GuardrailBlocked` | `rule, excerpt, blockedAt` | no status change (audit-only) |
| `CaseFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `SupportCaseRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailBlocks = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SupportCaseRow` mirrors `SupportCaseRecord` exactly. The UI fetches the full row via `GET /api/cases/{id}` and streams updates via `GET /api/cases/sse`.

The view declares ONE query: `getAllCases: SELECT * AS cases FROM support_case_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions

### `TriageTasks.java`

```java
public final class TriageTasks {
  public static final Task<CustomerInfo> GATHER_CUSTOMER_INFO = Task
      .name("Gather customer info")
      .description("Use lookupCustomerAccount and classifyIssue to collect structured customer and issue data")
      .resultConformsTo(CustomerInfo.class);

  public static final Task<IssueSummary> SUMMARIZE_ISSUE = Task
      .name("Summarize issue")
      .description("Distill the gathered customer info into a concise IssueSummary with category, urgency, problem statement, and key facts")
      .resultConformsTo(IssueSummary.class);

  private TriageTasks() {}
}
```

### `ExpertTasks.java`

```java
public final class ExpertTasks {
  public static final Task<Recommendation> COMPOSE_RECOMMENDATION = Task
      .name("Compose recommendation")
      .description("Use knowledge-base tools to find relevant articles, then compose a grounded Recommendation citing them")
      .resultConformsTo(Recommendation.class);

  private ExpertTasks() {}
}
```

Both companion Tasks classes are mandatory for their respective AutonomousAgents (Lesson 7).

## Tool registration

`IntakeTools` is registered ONLY on `TriageAgent` via `.tools(IntakeTools.class)` in `TriageAgent.definition()`. `KnowledgeBaseTools` is registered ONLY on `ExpertAgent` via `.tools(KnowledgeBaseTools.class)` in `ExpertAgent.definition()`. Cross-registration is intentionally forbidden; tool separation is a build-time property.
