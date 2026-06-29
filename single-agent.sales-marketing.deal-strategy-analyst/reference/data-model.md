# Data model — deal-strategy-analyst

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `StakeholderSignal` | `role` | `String` | no | Functional role, e.g. "Economic Buyer", "Champion", "Legal". |
| | `concern` | `String` | no | The specific concern this stakeholder has raised. |
| | `engagementLevel` | `String` | no | `"high"`, `"medium"`, `"low"`, or `"unknown"`. |
| `DealContext` | `dealId` | `String` | no | UUID minted by `DealEndpoint`. |
| | `dealName` | `String` | no | User-supplied deal label. |
| | `dealType` | `String` | no | `"enterprise-new"`, `"renewal-risk"`, or `"competitive"`. |
| | `rawContext` | `String` | no | Pre-sanitization context bundle body. Audit-only. |
| | `stakeholders` | `List<StakeholderSignal>` | no | Submitted list (1–N). |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedContext` | `redactedContext` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","person-name","confidential-company"]`. |
| `NextStep` | `stepId` | `String` | no | Short slug, e.g. `"re-engage-economic-buyer"`. |
| | `actionType` | `ActionType` | no | Enum value. |
| | `stakeholderRole` | `String` | no | MUST equal a role in the submitted stakeholder list. |
| | `rationale` | `String` | no | 1–3 sentences. |
| | `suggestedDeadline` | `String` | no | ISO-8601 date or relative string. |
| | `priority` | `int` | no | 1 = highest; unique across all steps in a recommendation. |
| `StrategyRecommendation` | `urgency` | `Urgency` | no | Enum value. |
| | `executiveSummary` | `String` | no | 2–4 sentences. |
| | `nextSteps` | `List<NextStep>` | no | 3–6 items ordered by priority ascending. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `RecommendationScorer` finished. |
| `Deal` (entity state) | `dealId` | `String` | no | — |
| | `context` | `Optional<DealContext>` | yes | Populated after `DealSubmitted`. |
| | `sanitized` | `Optional<SanitizedContext>` | yes | Populated after `ContextSanitized`. |
| | `recommendation` | `Optional<StrategyRecommendation>` | yes | Populated after `RecommendationRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `DealStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `DealSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Deal` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ActionType`: `FOLLOW_UP_EMAIL`, `SCHEDULE_CALL`, `SEND_PROPOSAL`, `REQUEST_INTRO`, `ESCALATE`, `ADDRESS_OBJECTION`, `PROVIDE_REFERENCE`.

`Urgency`: `CRITICAL`, `HIGH`, `NORMAL`.

`DealStatus`: `SUBMITTED`, `SANITIZED`, `ANALYSING`, `RECOMMENDATION_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`DealEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DealSubmitted` | `context` | → SUBMITTED |
| `ContextSanitized` | `sanitized` | → SANITIZED |
| `AnalysisStarted` | — | → ANALYSING |
| `RecommendationRecorded` | `recommendation` | → RECOMMENDATION_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `DealFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Deal.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`DealRow` mirrors `Deal` minus `context.rawContext` (the audit log keeps that). The UI fetches the raw context on demand via `GET /api/deals/{id}` and reads `context.rawContext` from the JSON.

The view declares ONE query: `getAllDeals: SELECT * AS deals FROM deal_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`DealTasks.java`)

```java
public final class DealTasks {
  public static final Task<StrategyRecommendation> ANALYSE_DEAL = Task
      .name("Analyse deal")
      .description("Read the attached deal context bundle and produce a StrategyRecommendation with prioritised next steps")
      .resultConformsTo(StrategyRecommendation.class);

  private DealTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
