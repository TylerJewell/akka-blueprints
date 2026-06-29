# Data model — guardrails-baseline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PolicyRule` | `ruleId` | `String` | no | Stable id supplied by the operator. |
| | `description` | `String` | no | What the agent must check. |
| | `category` | `String` | no | Policy domain (e.g., `hate-speech`, `financial-advice`, `privacy`). |
| `ModerationRequest` | `moderationId` | `String` | no | UUID minted by `ModerationEndpoint`. |
| | `messageTitle` | `String` | no | Operator-supplied label. |
| | `rawMessage` | `String` | no | Pre-sanitization message body. Audit-only. |
| | `rules` | `List<PolicyRule>` | no | Submitted list (1–N). |
| | `submittedBy` | `String` | no | Operator identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedMessage` | `redactedMessage` | `String` | no | PII stripped; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","ssn","payment-card-number","person-name"]`. |
| `RuleResult` | `ruleId` | `String` | no | MUST equal a submitted `ruleId`. |
| | `action` | `RuleAction` | no | Enum value. |
| | `confidence` | `double` | no | `0.0` (uncertain) to `1.0` (certain). |
| | `explanation` | `String` | no | Why the rule fired or did not fire. |
| `ModerationDecision` | `verdict` | `Verdict` | no | Enum value. |
| | `rationale` | `String` | no | 1–3 sentences. |
| | `ruleResults` | `List<RuleResult>` | no | One entry per submitted rule. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `AuditResult` | `score` | `int` | no | 1–5. |
| | `note` | `String` | no | One sentence. |
| | `auditedAt` | `Instant` | no | When `AuditScorer` finished. |
| `Moderation` (entity state) | `moderationId` | `String` | no | — |
| | `request` | `Optional<ModerationRequest>` | yes | Populated after `MessageSubmitted`. |
| | `sanitized` | `Optional<SanitizedMessage>` | yes | Populated after `MessageSanitized`. |
| | `decision` | `Optional<ModerationDecision>` | yes | Populated after `DecisionRecorded`. |
| | `audit` | `Optional<AuditResult>` | yes | Populated after `AuditCompleted`. |
| | `status` | `ModerationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Moderation` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`RuleAction`: `ALLOW`, `ESCALATE`, `BLOCK`.
`Verdict`: `ALLOW`, `ESCALATE`, `BLOCK`.
`ModerationStatus`: `SUBMITTED`, `SANITIZED`, `MODERATING`, `DECISION_RECORDED`, `AUDITED`, `FAILED`.

## Events (`ModerationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MessageSubmitted` | `request` | → SUBMITTED |
| `MessageSanitized` | `sanitized` | → SANITIZED |
| `ModerationStarted` | — | → MODERATING |
| `DecisionRecorded` | `decision` | → DECISION_RECORDED |
| `AuditCompleted` | `audit` | → AUDITED (terminal happy) |
| `ModerationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Moderation.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ModerationRow` mirrors `Moderation` minus `request.rawMessage` (the audit log keeps that). The UI fetches the raw message on demand via `GET /api/moderations/{id}` and reads `request.rawMessage` from the JSON.

The view declares ONE query: `getAllModerations: SELECT * AS moderations FROM moderation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ModerationTasks.java`)

```java
public final class ModerationTasks {
  public static final Task<ModerationDecision> MODERATE_MESSAGE = Task
      .name("Moderate message")
      .description("Evaluate the attached sanitized message against each submitted policy rule and produce a ModerationDecision")
      .resultConformsTo(ModerationDecision.class);

  private ModerationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
