# Data model — case-management-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundMessage` | `messageId` | `String` | no | UUID minted by `CaseEndpoint`. |
| | `customerId` | `String` | no | Customer identifier from the request. |
| | `existingCaseId` | `Optional<String>` | yes | Supplied when the message relates to an open case. |
| | `rawText` | `String` | no | Pre-sanitization message body. Audit-only. |
| | `channel` | `Channel` | no | Enum value. |
| | `receivedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedMessage` | `redactedText` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","payment-card","person-name","account-id"]`. |
| `CaseAction` | `actionType` | `ActionType` | no | Enum value. |
| | `targetCaseId` | `Optional<String>` | yes | Required for UPDATE, ESCALATE, CLOSE. |
| | `category` | `String` | no | e.g. `"billing"`, `"technical"`, `"account-access"`. |
| | `priority` | `Priority` | no | Enum value. |
| | `tier` | `Tier` | no | Enum value. |
| | `summary` | `String` | no | 1–2-sentence case description. |
| | `agentReasoning` | `String` | no | 1–2-sentence explanation of the chosen action. |
| `CrmRecord` | `caseId` | `String` | no | Stable case identifier. |
| | `customerId` | `String` | no | Customer identifier. |
| | `category` | `String` | no | As chosen by the agent. |
| | `priority` | `Priority` | no | Enum value. |
| | `tier` | `Tier` | no | Enum value. |
| | `status` | `CaseStatus` | no | Enum value. |
| | `summary` | `String` | no | Current case summary. |
| | `messageIds` | `List<String>` | no | All message ids that have touched this case. |
| | `openedAt` | `Instant` | no | When the case was first created. |
| | `resolvedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `ActionEvaluator` finished. |
| `CaseRecord` (entity state) | `caseId` | `String` | no | — |
| | `lastMessage` | `Optional<InboundMessage>` | yes | Populated after `MessageReceived`. |
| | `sanitizedMessage` | `Optional<SanitizedMessage>` | yes | Populated after `MessageSanitized`. |
| | `lastAction` | `Optional<CaseAction>` | yes | Populated after `ActionApplied`. |
| | `crmRecord` | `Optional<CrmRecord>` | yes | Populated after `ActionApplied`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `CaseStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageReceived` emitted. |
| | `resolvedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `CaseRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Channel`: `WEB_CHAT`, `EMAIL`, `PHONE_TRANSCRIPT`, `API`.

`ActionType`: `CREATE`, `UPDATE`, `ESCALATE`, `CLOSE`.

`Priority`: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.

`Tier`: `TIER_1`, `TIER_2`, `TIER_3`.

`CaseStatus`: `OPEN`, `IN_PROGRESS`, `ESCALATED`, `RESOLVED`, `FAILED`.

## Events (`CaseEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MessageReceived` | `message: InboundMessage` | → OPEN |
| `MessageSanitized` | `sanitized: SanitizedMessage` | (stays OPEN; sanitized field populated) |
| `AgentActing` | — | → IN_PROGRESS |
| `ActionApplied` | `action: CaseAction`, `crmRecord: CrmRecord` | OPEN → IN_PROGRESS; IN_PROGRESS → IN_PROGRESS (UPDATE) or ESCALATED (ESCALATE) or RESOLVED (CLOSE) |
| `EvaluationScored` | `eval: EvalResult` | (no status change; eval field populated) |
| `CaseFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `CaseRecord.initial("")` with all `Optional` fields as `Optional.empty()` and `status = OPEN`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`CaseRow` mirrors `CaseRecord` minus `lastMessage.rawText` (the audit log keeps that; the UI fetches it on demand via `GET /api/cases/{id}`).

The view declares ONE query: `getAllCases: SELECT * AS cases FROM case_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`CaseTasks.java`)

```java
public final class CaseTasks {
  public static final Task<CaseAction> HANDLE_MESSAGE = Task
      .name("Handle customer message")
      .description("Read the sanitized message and open-case context, choose an action type, and return a CaseAction")
      .resultConformsTo(CaseAction.class);

  private CaseTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
